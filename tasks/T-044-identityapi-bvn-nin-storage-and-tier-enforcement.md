# T-044: IdentityApi — add BVN/NIN storage and real KYC-tier evidence enforcement

Target repo: `Garbati.IdentityApi`. Branch: `feature/bvn-nin-storage-and-tier-enforcement`.

## Context

`CustomerIdentityData` has deliberately had no BVN/NIN column since T-016 (`docs/DESIGN.md` Appendix B,
`#TBD-4`) — left out pending a cryptography decision. That decision is now made: **garbati-platform D-26
(23 Oct 2026) — store BVN/NIN unencrypted for now, revisit encryption later.** This was decided after a
dedicated research task (`garbati-platform/research/ndpa-encryption-requirement.md`) read Nigeria's Data
Protection Act 2023 primary text directly and found no specific legal mandate requiring encryption — a
general, risk-based "appropriate measures" duty applies instead. Do not re-open that decision in this
task; it's settled, read `research/ndpa-encryption-requirement.md` only if you want the full reasoning.

This task does two things: add the actual columns, and close a real, separate gap found while scoping
this work — **`RecordKycDecisionAsync` currently lets any caller assign any `KycTier` to any customer with
zero check against what identity evidence actually exists on that customer's record.** Confirmed directly:
`KycTierPolicy.RequiredEvidence` (a configurable jsonb string list — decision #11, "configurable, not
hardcoded") is stored but **never read or enforced anywhere in the codebase** — grepped the full `src/`
tree for `RequiredEvidence` and found it only in migrations, the model snapshot, and the entity itself. A
caller can request `Tier3` for a customer with no evidence of anything on file and it will be granted.

Per `garbati-platform/research/nigerian-regulatory-requirements.md` Q1, the real tier definitions are:

| Tier | Requires |
|---|---|
| Tier 1 | Self-declared identity, plus **BVN or NIN** (at least one) |
| Tier 2 | Verified government ID, confirmed address, **BVN/NIN** (at least one — same cardinality as Tier 1, differs by document/verification level, not identity-number count) |
| Tier 3 | Full KYC: **both BVN and NIN**, additional ID, source-of-funds documentation |

This task closes the BVN/NIN-cardinality half of that gap specifically — not the whole `RequiredEvidence`
enforcement story. See "Explicitly out of scope" below for why `GovernmentId`/`SourceOfFunds` enforcement
is a separate, bigger, not-yet-started piece of work that this task does not touch.

## What to build

### 1. Add `Bvn`/`Nin` to `CustomerIdentityData`

- Two new nullable `string` columns on `Entities/CustomerIdentityData.cs`: `Bvn`, `Nin`. Unencrypted, per
  D-26 — do not add any encryption/KMS code; that is a separate, not-yet-decided follow-up.
- EF Core migration. Additive only — no existing column changes, no data migration needed (every existing
  row simply gets `NULL` for both new columns).
- Format: both BVN and NIN are 11-digit numeric strings in Nigeria (stable, well-known identifier
  formats — not a legal nuance needing further compliance confirmation). Validate exactly that: non-null
  and non-blank means exactly 11 characters, all ASCII digits. Reject anything else with a clear 400
  message naming which field failed and why (e.g., `"Bvn must be exactly 11 digits."`).

### 2. A new endpoint to submit BVN/NIN — set-once, not freely editable

Identity numbers are not like an email or phone number: they shouldn't be casually overwritten once set
(a channel that lets someone silently swap a stored BVN/NIN after the fact is a real fraud vector). Build
this as its own endpoint, separate from the existing `PATCH /v1/customers/{customerId}/details`
(`UpdateDetailsAsync`, T-035) which handles `FullName`/`Email`/`PhoneNumber` and is correctly *not* the
right place for set-once semantics.

- **`PATCH /v1/customers/{customerId}/identity-numbers`**, scope `IdentityScopes.KycManage` (the existing
  `identity.kyc.manage` scope — this is part of the KYC evidence pipeline, not general contact-detail
  editing; no new scope needed).
- Request: `SubmitIdentityNumbersRequest(string? Bvn, string? Nin)` — at least one of the two must be
  supplied (reject a request with both null, same pattern `UpdateCustomerDetailsRequestValidator` already
  uses for "at least one field supplied").
- **Set-once rule:** for each of `Bvn`/`Nin` individually —
  - If the stored value is currently `null` and the request supplies a value: set it (after format
    validation).
  - If the stored value is already set and the request supplies the **same** value: no-op, return 200
    (idempotent — resubmitting your own correct BVN twice should not be an error).
  - If the stored value is already set and the request supplies a **different** value: reject with
    **409** and a clear message (e.g., `"Bvn is already set for this customer and cannot be changed via
    this endpoint."`). There is deliberately no "admin override" path in this task — if a real correction
    process is ever needed, that's a separate, more deliberate follow-up, not a side effect of this one.
  - A request field left `null` (not supplied) never touches the corresponding stored value, same
    "null means leave alone" convention `UpdateCustomerDetailsRequest` already uses.
- Audit: record an `AuditEvent` (reuse the existing `AuditLogger`/`AuditEvent` pattern) on any actual
  change, action name `IdentityNumberSubmitted` (add to `AuditActions`) — **the `Details` string must
  never contain the raw BVN/NIN value.** State only which field(s) changed (e.g.,
  `"Bvn set"`/`"Nin set"`), the same discipline `RecordKycDecisionAsync`'s own `Details` string already
  follows for `KycStatus`/`KycTier` (which are never sensitive) — do not copy that pattern carelessly into
  a field where the value itself is exactly what must never be logged (`ARCHITECTURE.md` Section 5: "Logs
  never contain identity numbers (BVN/NIN)" — this applies to audit `Details` text too, not only to
  Serilog request logs).
- Response: return the existing `CustomerResponse` (unchanged shape — it already excludes every
  `CustomerIdentityData` field, including the new ones; do **not** add `Bvn`/`Nin` to `CustomerResponse`
  or any other response DTO anywhere. The redacted-view guarantee in `GET /v1/customers/{customerId}`'s
  own doc comment — "never raw CustomerIdentityData fields" — must stay true after this task, not just
  before it).

### 3. Enforce BVN/NIN cardinality when `RecordKycDecisionAsync` grants a tier

In `Services/Implementations/CustomerService.cs`, `RecordKycDecisionAsync`, **before** the line that
assigns `customer.KycTier = request.KycTier` (only reached when `newStatus == KycStatus.Verified`):

- Add two new evidence tokens to the vocabulary: **`AtLeastOneIdentityNumber`** (satisfied when
  `CustomerIdentityData.Bvn` or `.Nin` is non-null) and **`BothIdentityNumbers`** (satisfied when both are
  non-null). These are additions to the existing, already-configurable `KycTierPolicy.RequiredEvidence`
  jsonb vocabulary (decision #11) — not a new hardcoded rule bypassing that mechanism.
- Update the three existing placeholder rows in the `SeedPlaceholderKycTierPolicies` migration data (via a
  new migration that updates them, not by editing the old migration file) to include the real BVN/NIN
  cardinality from the table above: `Tier1` and `Tier2` gain `"AtLeastOneIdentityNumber"`; `Tier3` gains
  `"BothIdentityNumbers"` (replacing nothing — add to the existing `RequiredEvidence` lists, which already
  contain `SelfDeclaredIdentity`/`GovernmentId`/`SourceOfFunds` placeholder tokens). These rows are still
  explicitly "placeholder data, not real Nigerian tier limits" per the original migration's own comment —
  you are making the BVN/NIN cardinality *part* of that placeholder accurately reflect Q1's real tier
  definitions, not asserting a new, independently-confirmed legal fact.
- **Enforcement scope — read carefully, this is deliberately narrow:** when granting a tier, look up the
  currently-effective `KycTierPolicy` for `(customer.JurisdictionCode, request.KycTier)` and check **only**
  whether `AtLeastOneIdentityNumber`/`BothIdentityNumbers` (whichever is present in that policy's
  `RequiredEvidence`) is satisfied. **Do not** attempt to enforce `SelfDeclaredIdentity`/`GovernmentId`/
  `SourceOfFunds` — those have no backing data anywhere in the system yet (no column, no document-storage
  mechanism exists for any of them), and trying to enforce them now would make it permanently impossible
  to ever grant Tier 2/3 until a much bigger, separate piece of work (document evidence storage) is built.
  Leave those three tokens exactly as unenforced as they are today — this task closes the BVN/NIN gap
  specifically, not the full `RequiredEvidence` story.
- If the required cardinality is not met: reject the whole `RecordKycDecisionAsync` call with **422** and
  a stable reason (e.g., `"KycTier Tier3 requires both Bvn and Nin to be on file for this customer."`) —
  do **not** record the `KycVerification` row or change `customer.KycStatus` at all in this case; the
  entire decision fails atomically, the same "whole operation fails together" pattern already used
  elsewhere in this codebase (e.g., `AccountOpeningService`'s step failures in the orchestrator, or a
  failed chain reversal in LedgerApi) rather than a partial, confusing half-applied state.
- A `KycStatus` decision that is **not** `Verified` (e.g., `Rejected`, `Expired`, back to `Pending`) is
  never subject to this check — the check only ever gates *granting* a tier, exactly matching the existing
  `if (newStatus == KycStatus.Verified)` guard already in the method.
- No matching `KycTierPolicy` row for the requested `(JurisdictionCode, TierCode)` at all is a different,
  pre-existing failure mode — check how the method currently behaves in that case (if it doesn't check
  today, that's fine, out of scope for this task; don't invent new behavior for it unless your new
  enforcement code would otherwise throw an unhandled exception on a missing row, in which case handle it
  the same way as the cardinality-not-met case, 422, not a 500).

## Explicitly out of scope

- **Any real external BVN/NIN verification** (calling NIBSS, a third-party KYC provider, or any other
  external API to confirm the number is authentic/matches the claimed identity). This task only stores a
  self-/ops-submitted value and checks its *presence*, never its *authenticity*. Adding a real external
  verification integration is both a new dependency and a real design task of its own — not decided, not
  started, not this task's job.
- **Encryption at rest.** Decided (D-26): unencrypted for now. Do not add any KMS/encryption code.
- **Retention/purge policy** for how long BVN/NIN is kept after account closure. Still explicitly open
  (`garbati-platform/research/nigerian-regulatory-requirements.md` Q2/Q3, 5 vs. 6 years unresolved) — no
  purge job, no deletion logic, nothing time-based. Storing it indefinitely for now is the status quo for
  every other field on this table too; this task doesn't change that posture either way.
- **`GovernmentId`/`SourceOfFunds` evidence enforcement.** Pre-existing gap (confirmed: never enforced
  today), not created or worsened by this task, not this task's job to fix.
- **Any change to `UpdateCustomerDetailsRequest`/`UpdateDetailsAsync`** (T-035's endpoint) — BVN/NIN get
  their own endpoint with different (set-once) semantics; do not add them there instead.

## Tests to write

- Migration applies cleanly; existing rows get `NULL` for both new columns (no data loss, no default
  value invented).
- Format validation: reject a `Bvn`/`Nin` that isn't exactly 11 digits (too short, too long, contains a
  letter or symbol); accept a valid 11-digit value.
- Set-once behavior: first submission of a new value succeeds; resubmitting the *same* value succeeds as
  a no-op (verify no duplicate audit row is written on the no-op case, or that it's written but clearly
  marked as a no-op — your call, document which you chose and why); submitting a *different* value when
  one is already stored returns 409 and the stored value is provably unchanged (query the database
  directly to confirm, the same way this platform's QA process has verified "nothing changed" claims
  before — don't just trust the response code).
- Tier-enforcement: granting `Tier1`/`Tier2` succeeds when at least one of `Bvn`/`Nin` is on file, fails
  (422) when neither is; granting `Tier3` succeeds only when **both** are on file, fails (422) when only
  one is present. A non-`Verified` decision (e.g., `Rejected`) never triggers this check regardless of
  what's on file.
- Audit content: query the actual `AuditEvent`/`audit_events` row written by a successful
  `IdentityNumberSubmitted` action and assert its `Details` string does **not** contain the literal BVN/
  NIN value submitted in the test — a real, direct assertion against the stored string, not just "no
  exception was thrown."
- `CustomerResponse` (from any endpoint that returns it, including the new one) never contains a `Bvn` or
  `Nin` property — this can be a compile-time fact (don't add the properties) but add one test that
  deserializes a real response and asserts the JSON has no such keys, so a future change can't
  accidentally reintroduce them without a test actually catching it.

## PR

Open a pull request from `feature/bvn-nin-storage-and-tier-enforcement` against `main` on
`AbubakarImam/Garbati.IdentityApi` (branch-protected — get `test`/`docker` CI green first). Describe
exactly what changed, the set-once design choice and why, and the deliberately narrow enforcement scope
(BVN/NIN cardinality only, not the full `RequiredEvidence` vocabulary). Do not merge — only the owner
merges.
