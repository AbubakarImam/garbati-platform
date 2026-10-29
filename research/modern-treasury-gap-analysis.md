# Modern Treasury gap analysis — what GARBATI has, lacks, or differs on by design (T-055)

**Task:** T-055. **Status:** Research only — no code, no design decision, no schema, no recommendation to
build anything found missing here. This brief reads Modern Treasury's real documentation (`docs.moderntreasury.com`)
across its actual current structure and checks each concept found against GARBATI's actual current
implementation — the real service code, not GARBATI's own documentation about itself taken at face value,
matching this project's "verify, don't trust" discipline (`AGENTS.md`). It builds on, and does not
re-research, `research/reconciliation-nibss-integration-patterns.md` (T-054), which already read Modern
Treasury's own reconciliation-definition page directly and found GARBATI's three named reconciliation gaps
are not nostro/vostro-style reconciliation because GARBATI has no live external-rail leg yet. This brief
confirms that same precondition independently, from a different angle (Expected Payments, Returns), and
does not repeat T-054's own findings.

**The framing this brief holds itself to throughout:** Modern Treasury is infrastructure other companies
integrate with to add payment-ops/ledger capability to their own product. GARBATI is a vertically-integrated
core-banking platform built *for* Sharia-compliant microfinance institutions, not a layer other companies
build on top of. Many of Modern Treasury's concepts exist specifically because its customers don't own a
banking license or a core ledger of their own — Virtual Accounts and Counterparties are the clearest
examples. GARBATI, as the institution's own core system, doesn't have the problem those features solve.
That is a genuine "different by design" finding, not a euphemism for an unexamined gap — each one below is
checked on its own terms, not marked missing by default.

---

## Plain-language summary (read this first)

Modern Treasury's documentation turned out to be organized differently than this task file's original guess
(see "Site structure actually found" below) — it is split into a Payments product (counterparties, payment
orders, virtual accounts, expected payments, returns, reversals, approval rules, webhooks) and a separate
Ledgers product (ledger/ledger account/ledger transaction/ledger entry, ledgerable associations), each with
its own reference docs. Every concept named in the task file was found on the real site and read directly;
nothing in the task's starting list turned out to be wrong, though several concepts (Reversal vs. Return,
Legal Entity, Approval Rules) were richer or more specifically scoped in the real docs than the task file's
one-line descriptions suggested.

The honest overall shape of the comparison: **GARBATI has a real, built equivalent for every core
double-entry and authorization-control concept** (ledger accounts/transactions/entries, idempotency,
reversal, multi-party approval) — confirmed against actual code, not just design docs. **GARBATI is
genuinely missing the concepts that exist specifically because Modern Treasury's customers deal with the
outside world** (an external payee/payer with real bank details, a declared incoming payment awaited from
outside, a documented event-notification system for external integrators) — and in every one of those cases,
the reason is the same, already-established fact from T-054: **GARBATI has no live external-rail
integration (NIBSS or otherwise) yet.** None of these are urgent gaps; they become relevant the day GARBATI
has an external counterparty to deal with, not before. And two concepts — Virtual Accounts and (for GARBATI's
customer-facing side) the Legal Entity/Counterparty split — are missing specifically **because** GARBATI is
a different kind of product, not because anyone forgot to build them.

---

## Site structure actually found (confirmed by direct fetch, not assumed from the task file)

Modern Treasury's documentation is organized into at least two distinct product surfaces with separate
reference trees, confirmed by fetching pages directly:

- **A Payments guide tree** (`docs.moderntreasury.com/payments/...` and `.../platform/reference/...`),
  covering: Getting Started/Sandbox, Customer Onboarding (Legal Entities), Managing Internal Accounts,
  Payments (Originating — ACH/wire/RTP/stablecoin/check/push-to-card, Receiving — tracking, reconciliation,
  Creating Counterparties — bank-account verification, bulk import, hosted onboarding), Workflows &
  Reporting (account groups, automatic sweeps, approval rules, payment templates), and "Bring Your Own Bank"
  (bank integration management, Virtual Accounts, bank-specific rails, FX, payment actions, bank holds).
- **A Ledgers reference tree** (`docs.moderntreasury.com/platform/reference/ledger-*`), covering Ledger
  Accounts, Ledger Transactions, Ledger Entries, and how a ledger transaction binds to another Modern
  Treasury object via `ledgerable_type`/`ledgerable_id`.
- A general top-level "Ledgers overview" landing page (guessed at by this research as
  `/ledgers/docs/61-ledgers-overview`, matching the task file's framing) returned a 404 on direct fetch —
  this specific URL does not exist on the current site; the ledger reference pages above were reached
  instead via search and read directly, and are treated as authoritative for this brief. This is stated
  plainly as an access limitation on one specific guessed URL, not a claim that Modern Treasury has no
  ledgers overview content at all.
- Every other page cited below (Payment Order, Counterparty, Virtual Account, Expected Payment, Return,
  Reversal, Approval Rules, Legal Entity, Webhooks, Idempotent Requests, Ledger Account/Transaction/Entry)
  was fetched and read directly, not taken from a search-result summary alone, except where explicitly
  flagged otherwise below.

---

## Concept-by-concept comparison

### 1. Ledgers core model — Ledger Account, Ledger Transaction, Ledger Entry

**Verdict: GARBATI has a real equivalent, confirmed built — with one honest structural difference worth naming.**

- Modern Treasury: a `ledger_account` has a `normal_balance` (debit-normal for assets, credit-normal for
  liabilities) and three balance tiers — **pending**, **posted**, and **available** — tracked per account and
  per entry. A `ledger_transaction` requires at least one debit and one credit entry summing to the same
  amount, has status `pending` → `posted` (immutable) → `archived`, and becomes immutable once posted. A
  `ledger_entry` carries `direction` (debit/credit), amount, and the account's resulting balances.
  [Ledger Account Object](https://docs.moderntreasury.com/platform/reference/ledger-account-object);
  [Ledger Transaction Object](https://docs.moderntreasury.com/platform/reference/ledger-transaction-object);
  [Ledger Entry Object](https://docs.moderntreasury.com/platform/reference/ledger-entry-object)
- GARBATI's `LedgerApi` (read directly: `Entities/Account.cs`, `Entities/Transaction.cs`,
  `Entities/LedgerEntry.cs`, `Entities/EntryType.cs`, `Entities/TransactionStatus.cs`) has the same shape at
  its core: `Account` (with `AccountType`, `AccountClass` = `Customer`/`System`, `AccountStatus` =
  `Active`/`Frozen`/`Blocked`) holds `LedgerEntry` rows, each with `EntryType` = `Debit`/`Credit` and an
  `Amount`; a `Transaction` requires exactly one debit and one credit entry (the tested, enforced rule); every
  posting happens inside one database transaction (atomicity by database guarantee, not by status field).
- **The honest difference, confirmed by reading `AccountService.GetBalanceAsync`**: GARBATI computes a
  single derived balance by summing all of an account's entries on read (`dbContext.LedgerEntries.Where(x =>
  x.AccountId == account.Id)...`) — there is no stored pending/posted/available split. `Transaction.Status`
  has three values (`Pending`, `Success`, `Failed`), not Modern Treasury's four (`pending`/`posted`/`archived`
  plus the fact that Modern Treasury's `pending` can persist for real, observable time). This difference is
  not an oversight: Modern Treasury's three-tier balance model exists because its ledger entries can
  genuinely sit in `pending` for hours or days while an external rail (ACH, wire) actually clears — GARBATI's
  `LedgerApi` posts everything synchronously inside one all-or-nothing database transaction (decision #45,
  `ARCHITECTURE.md` Section 12.2) because it has no external rail yet; there is currently nothing for a
  GARBATI transaction to be "pending" *against*. This is the same underlying fact T-054 already established
  (no live NIBSS leg) showing up again from the ledger-model side.
- Modern Treasury also has a `Ledger` object as an explicit top-level container (a customer can run multiple
  ledgers). GARBATI has no equivalent container entity — there is one implicit ledger per `LedgerApi`
  deployment. This maps onto `ARCHITECTURE.md` Section 9's still-open multi-tenancy question (T-050): if
  GARBATI is ever deployed as one shared platform serving several institutions rather than one deployment per
  institution, an explicit ledger/tenant container (closer to Modern Treasury's `Ledger` object) would likely
  become necessary. Today, with one deployment per institution assumed, it is not needed — **different by
  design given GARBATI's current single-tenant assumption**, not a gap, though worth flagging as the same
  open question T-050 already surfaced from a different direction.

### 2. Ledgerable associations (`ledgerable_type`/`ledgerable_id`)

**Verdict: different by design, with a real architectural reason — but with one honest practical limitation worth naming.**

- Modern Treasury lets a `ledger_transaction` (and a `ledger_account`) carry a polymorphic pointer to another
  of its own business objects — `ledgerable_type` one of `payment_order`, `incoming_payment_detail`,
  `expected_payment`, `return`, or `reversal` (for transactions), or `external_account`/`internal_account`/
  `virtual_account` (for accounts) — and Modern Treasury then automatically propagates status changes from the
  linked object onto the ledger transaction. [Ledger Transaction Object](https://docs.moderntreasury.com/platform/reference/ledger-transaction-object)
- GARBATI's `LedgerApi` has nothing resembling a generic, polymorphic "attach this transaction to an
  arbitrary external business object" field. What it has instead, read directly in `Entities/Transaction.cs`
  and `Entities/TransactionChain.cs`, is narrower and purpose-built: `ChainId`/`Sequence`/`Purpose`
  (`LegPurpose`: `Principal`/`Fee`/`Tax` only) ties a leg to its chain, and a chain's own `IdempotencyKey` is
  the thing a caller correlates against.
- **This is architecturally deliberate, not an oversight.** `ARCHITECTURE.md` decision #34 ("LedgerApi calls
  nobody") and P-5 (no shared database, no cross-service foreign keys) together rule out `LedgerApi` storing a
  typed pointer into another service's own entity (an `AccountOpening` row in `OrchestratorApi`, say) the way
  Modern Treasury's single-database monolith can point from a ledger transaction to its own `payment_order`
  row. GARBATI's services correlate across that boundary by passing the same idempotency key and
  `X-Correlation-ID` through every call (`ARCHITECTURE.md` Section 5) instead of storing a foreign-key-shaped
  association inside the ledger itself.
- **The honest practical cost of this choice**, not papered over: Modern Treasury's model lets you ask "show
  me the ledger transaction for payment order X" with one join inside its own database. GARBATI cannot do the
  equivalent today — finding the ledger transaction(s) behind one `AccountOpening` or one `Transfer` means
  correlating by idempotency key/reference across two services' separate databases at query time, not a stored
  link. That is a real, if minor, operational inconvenience (relevant to building any future ops-console
  "show me everything about this operation" view) that follows directly from P-5, not a design gap worth
  reopening — P-5 was a deliberate, already-ratified choice (independent service databases) with this as one
  of its known costs.

### 3. Payment Orders and their lifecycle

**Verdict: mixed — GARBATI has a real, much thinner equivalent for the money-movement part; the approval-gate part has real equivalents built at a different layer; the scheduling/external-rail part is genuinely missing, but already a known, named gap on GARBATI's own roadmap, not a new finding.**

- Modern Treasury's `payment_order` is an instruction to move money via a named external rail (ACH, wire,
  RTP, stablecoin, check, card), with a rich, asynchronous lifecycle: `needs_approval` → `approved`/`denied` →
  `processing` → `sent` → `completed`/`returned`/`reversed`/`failed`, plus `held`/`cancelled`/`stopped`. It
  carries `effective_date` (when it posts), `priority` (same-day vs. normal), and links to a
  `ledger_transaction_id` once posted. [Payment Order Object](https://docs.moderntreasury.com/platform/reference/payment-order-object)
- GARBATI's closest equivalent, `OrchestratorApi`'s `TransferService` (read directly), is a **synchronous,
  single-call, intra-platform book transfer**: request authorization decision from `IdentityApi`, then post
  immediately to `LedgerApi` — no persisted "transfer" entity at all (confirmed: the code's own comment says
  "No new persistent entity for a transfer itself ... both downstream calls are already independently
  idempotent"), no `needs_approval`/`held`/`sent` states, no scheduling, no external rail. This is a
  materially thinner model, and honestly so — GARBATI has nothing today that moves money anywhere outside its
  own `LedgerApi`.
- GARBATI's `AccountOpening` entity (read directly: `Entities/AccountOpening.cs`,
  `Entities/AccountOpeningStep.cs`, `Entities/AccountOpeningStatus.cs`) is the one real, persisted,
  multi-step, resumable operation state machine GARBATI has — `CurrentStep` (`CreateCustomer` → `VerifyKyc` →
  `CreateLedgerAccount` → `AttachMandate` → `BindProduct` → `Complete`) and `Status`
  (`InProgress`/`Complete`/`Failed`) — genuinely comparable in *shape* to a payment order's status machine,
  just for a different operation (opening an account, not moving money to an external rail).
  **This is a real, confirmed-built equivalent for "a multi-step operation with durable, resumable state,"**
  just applied to a different use case than Modern Treasury's payment orders.
- **The approval-gate part of a payment order's lifecycle (`needs_approval`/`approved`/`denied`) has a real
  GARBATI equivalent, confirmed built, just split across two services rather than living in one object**: see
  Section 8 below (Approval Rules). This is marked there, not here, to avoid double-counting.
- **Scheduling (`effective_date`, recurring payments) and submission to an actual external rail are genuinely
  missing** — but this is not a new finding. `ARCHITECTURE.md`'s own build order already names `ScheduleApi`
  (standing orders) as "later tranche, not designed yet," and names the entire Channel/Gateway and real NIBSS
  integration as future work. This brief's contribution is only confirming that Modern Treasury's Payment
  Order object is the concrete, named shape that future work would eventually need to approximate some of —
  not a new gap this research discovered.

### 4. Counterparties

**Verdict: genuinely missing, worth considering — confirmed by code search, not assumed.**

- Modern Treasury: a counterparty is "an entity outside of your organization that you want to send or receive
  money from" — a person or business with its own linked external bank accounts (account/routing details),
  created before any payment order can target it.
  [Counterparty Object](https://docs.moderntreasury.com/platform/reference/counterparty-object)
- A repository-wide search across every GARBATI service (`LedgerApi`, `Garbati.IdentityApi`,
  `Garbati.OrchestratorApi`, `Garbati.ProductApi`, `Garbati.StaffIdentityApi`, `Garbati.OpsApi`) for
  `Counterparty`, `ExternalAccount`, `BankAccountNumber`, `RoutingNumber` returned **zero matches** in any
  `.cs` file. Confirmed directly, not inferred from a design doc: GARBATI has no concept, anywhere, of an
  external party with its own bank details that a transfer could target. Every `DebitAccountId`/
  `CreditAccountId` GARBATI's `TransferRequest` accepts (confirmed in `OrchestratorApi`'s
  `TransferService.cs`) must already be a `LedgerApi` account number inside GARBATI's own platform.
- **Why this is worth considering rather than urgent**: it is the direct structural consequence of the same
  fact T-054 already established — GARBATI has no live external-rail integration yet, so every transfer today
  is necessarily GARBATI-internal (same institution, same ledger). The day GARBATI needs to move money to or
  from an account outside its own platform (a customer paying a utility company, receiving a salary via
  NIBSS from another bank), there is currently no place to record *who that external party is*, what bank
  they're at, or how their account details were verified (Modern Treasury's own flow requires micro-deposit
  or Plaid-style verification before trusting a counterparty's bank details). This is a real, specific,
  concrete future need, not a generic "Modern Treasury has it so GARBATI should" — but it is explicitly not
  proposed for building here, per this task's scope.

### 5. Virtual Accounts

**Verdict: different by design, with the real reason stated plainly.**

- Modern Treasury: "A `virtual_account` represents a subset of the transactions in an [Internal Account]" —
  it lets a single real bank account be subdivided and attributed to many end users, so a company that
  doesn't hold its own banking license can still track per-customer balances without opening a real bank
  account per customer. [Virtual Account Object](https://docs.moderntreasury.com/platform/reference/virtual-account-object)
- GARBATI has no equivalent, and the reason is structural, not an oversight: GARBATI's `LedgerApi` already
  gives every retail customer a **real, directly-owned `Account` row** the moment an account is opened
  (confirmed: `AccountOpening`'s `CreateLedgerAccount` step, `LedgerApi`'s `POST /api/accounts`). GARBATI
  *is* the institution's own core ledger — there is no single pooled "Internal Account" sitting between
  GARBATI and a real bank that would need subdividing. Virtual Accounts solve a problem that exists
  specifically for a company sitting on top of someone else's bank account; GARBATI's customers are the
  core-banking records of record already. This is the clearest example in this whole brief of the task
  file's own warning: a Modern Treasury feature that is simply solving a different product's problem.

### 6. Expected Payments

**Verdict: genuinely missing, worth considering — and the single most directly relevant Modern Treasury concept to this platform's own already-named future reconciliation need.**

- Modern Treasury: an `expected_payment` is a declared, awaited incoming payment — `amount_lower_bound`/
  `amount_upper_bound`, `date_lower_bound`/`date_upper_bound`, `direction`, `type`, an associated
  `counterparty_id` — that Modern Treasury automatically matches against actual incoming bank transactions
  (typically ingested via a morning BAI2 file), with its own lifecycle:
  `created` → `tentatively_reconciled`/`partially_reconciled`/`reconciled`, `overdue` (fired 6am Pacific once
  `date_upper_bound` passes), `unreconciled`, `archived`.
  [Expected Payment object / webhooks](https://docs.moderntreasury.com/platform/reference/expected-payments)
- GARBATI has nothing resembling this anywhere — no entity, no table, no concept of "we are expecting an
  amount in this range, from this party, by this date" for an incoming payment. This was confirmed by the
  absence of any such concept across the code already read for this brief and by the complete absence of
  "Counterparty" (Section 4) — without an external party to expect a payment *from*, there is nowhere to hang
  an Expected Payment concept today either.
- **Why this is the most relevant single finding to flag for the future**: `research/reconciliation-nibss-integration-patterns.md`
  Q5 already concluded that a dedicated reconciliation service "becomes a much stronger case the day real
  NIBSS settlement exists" but did not name a concrete mechanism for what that service would actually do.
  Modern Treasury's Expected Payment object is exactly that mechanism, named and shaped concretely: a
  tolerance-windowed (amount range + date range), fuzzy-matched declaration of an awaited external payment,
  with its own explicit lifecycle states for partial/tentative/overdue matches — precisely the "fuzzy matching
  against an outside party's record" T-054 Q2/Q4 described in the abstract as the thing GARBATI's current
  internal checks don't need yet, but a future NIBSS-settlement reconciliation would. This brief does not
  propose building it — it only names the real, concrete, already-productized shape a future design could
  start from, which is new information T-054 did not have.

### 7. Returns vs. Reversals — two genuinely distinct Modern Treasury objects, compared separately against GARBATI's one reversal mechanism

**Verdict: GARBATI has a real, confirmed-built equivalent to Modern Treasury's Reversal concept; it is correctly missing the Return concept, for the same "no live rail yet" reason as Sections 4 and 6 — not a gap.**

- Modern Treasury draws a real, documented distinction between two different objects, confirmed by reading
  both pages directly:
  - **Return**: a rejection of a payment order **initiated by the bank/rail after the fact** — e.g. an ACH
    return with reason code `R01` (insufficient funds) — something that happens *to* a payment Modern
    Treasury already sent, reported back by the receiving or originating bank, outside Modern Treasury's
    direct control. [Return Object](https://docs.moderntreasury.com/platform/reference/return-object)
  - **Reversal**: a mechanism to **actively undo** a payment order, initiated by the integrator itself ("only
    used when integrating your own bank accounts with Modern Treasury... your internal account must support
    ACH origination in the opposite direction"). [Reversal Object](https://docs.moderntreasury.com/platform/reference/reversals)
- GARBATI's own reversal flow — read directly across all three services, not merely cited from
  `ARCHITECTURE.md`'s own description of itself — is real and built end to end: `Garbati.IdentityApi/docs/DESIGN.md`
  Section 8 (decisions #30–#35) defines a trusted-ops-channel-only reversal authorization that bypasses the
  account's own mandate entirely (decision #31, "ops override"), with five fixed reason codes reusing ISO
  20022/card-network vocabulary (`DUPL`/`TECH`/`FRAD`/`CUST`/`UNAUTH`, decision #32) and an outcome-tracking
  entity (`ReversalOutcomes`, `Completed`/`WriteOff`, decision #34). `Garbati.OrchestratorApi`'s
  `ReversalService.cs` (read directly) actually calls `IdentityApi` for authorization, then `LedgerApi` to
  reverse, and reports the real outcome back — including deliberately distinguishing a confirmed `WriteOff`
  (LedgerApi said 404/422, the reversal could not happen) from an `Unconfirmed` one (a timeout or 500 where
  nothing is actually known), a level of honesty about uncertain outcomes Modern Treasury's own public
  Reversal object documentation does not describe needing. `LedgerApi` decision #45 Section 12.8 (read
  directly in `ARCHITECTURE.md`) implements the actual posting-side mechanics: one reversing leg per original
  leg, last leg first, all-or-nothing. **This is GARBATI's real equivalent to Modern Treasury's Reversal
  concept, confirmed built across all three services, not just designed.**
- GARBATI has nothing resembling Modern Treasury's Return concept — and correctly so, for exactly the same
  reason Sections 4 and 6 already established: a Return is, by definition, a rail or counterparty bank telling
  *you* your payment was rejected after you sent it. GARBATI has no external rail to send a payment to yet,
  so there is nothing that could generate a Return. This is "not yet relevant," matching the task file's own
  prediction, not a gap.

### 8. Approval Rules (Modern Treasury's maker-checker for releasing payments)

**Verdict: GARBATI has a real equivalent, confirmed built in two separate places, serving two different actors — not one unified mechanism, but functionally comparable.**

- Modern Treasury: administrators configure conditional rules (AND within a block, OR across blocks) that
  require a named group of staff users to review a payment order or external account before it can proceed;
  all reviewers approving moves a payment order to `approved` (sent to the bank); any denial moves it to
  `denied`. A reviewer cannot approve something they created, and a single user cannot satisfy a
  multi-reviewer requirement alone. [Approval Rules Overview](https://docs.moderntreasury.com/payments/docs/approval-rules-overview)
- GARBATI has two real, separately-built mechanisms serving an analogous "more than one party must agree
  before this happens" purpose, confirmed by reading actual entity code, not ARCHITECTURE.md's own summary of
  itself:
  - **Customer-side**: `Garbati.IdentityApi`'s joint-account M-of-N approval (decisions #19–#25, confirmed in
    `docs/DESIGN.md`) — multiple mandate holders on one account, a configurable "any M of N" threshold, a
    1-hour pending-approval expiry, immediate voiding of a stale partial approval on an approver's status
    change, and single-use/request-bound approvals. This governs whether **a customer-initiated transaction**
    on a jointly-held account may proceed — a different trigger than Modern Treasury's rules (which gate a
    staff member releasing a payment to a bank), but the same underlying "collect enough independent consents
    before this is allowed to happen" shape.
  - **Staff-side**: `Garbati.OpsApi`'s maker-checker — confirmed as real code, not just a design claim, by
    directly finding `Entities/OpsReversalRequest.cs`, `Entities/OpsReversalRequestStatus.cs`,
    `Entities/OpsCustomerEditRequest.cs`, and `Entities/OpsCustomerEditRequestStatus.cs` in the actual
    `Garbati.OpsApi` repository. This is the closer structural analogue to Modern Treasury's Approval Rules
    (internal staff reviewing a sensitive action before it executes), though this research did not read
    `OpsApi`'s full workflow logic in depth (out of this brief's scope) — only confirmed the entities
    genuinely exist as built code, not merely described in a design doc.
- Marked as a real equivalent because the underlying capability (a sensitive action blocked pending
  independent, auditable consent) is confirmed to exist and be built in both the customer-facing and
  staff-facing cases — just organized around GARBATI's own actors (joint mandate holders; ops staff) rather
  than Modern Treasury's single staff-only reviewer-group model, because GARBATI's reversal/account-edit
  actions are staff actions and GARBATI's joint-account transfers are customer actions, two genuinely
  different populations Modern Treasury's single B2B staff-tool product doesn't need to distinguish.

### 9. Legal Entities (KYC/KYB identity data)

**Verdict: GARBATI has a real equivalent for its own customers; missing the counterparty-side half, which is the same underlying gap as Section 4, not a separate one.**

- Modern Treasury: a `legal_entity` is a person or business with distinct legal rights
  (`legal_entity_type`: `individual` or `business`), carrying KYC/KYB fields (name, DOB, addresses,
  identifications for individuals; legal structure, formation date, expected activity volume for
  businesses), shareable with partner banks for their own KYC/KYB checks, and attachable both to a
  Counterparty and to the integrator's own Internal Account. [Legal Entities](https://docs.moderntreasury.com/platform/reference/legal-entities)
- GARBATI's `Customer`/`CustomerIdentityData`/`KycVerification`/`KycTierPolicy` (confirmed in
  `Garbati.IdentityApi`, built and merged per decisions #11/#12/#16, and extended with real BVN/NIN storage
  per D-26) is a real, built equivalent **for GARBATI's own retail customers** — the people who hold accounts
  directly on GARBATI's own ledger.
- What's missing is specifically the half Modern Treasury's Legal Entity also covers that GARBATI has no use
  for yet: identity data **about an external counterparty** (the person or business a customer is sending
  money to or receiving from outside GARBATI's own platform). This is not a new, separate gap — it is the
  identity-data side of the exact same missing concept named in Section 4 (Counterparties), surfacing again
  here because Modern Treasury's own docs attach Legal Entity data to both sides (its own customer's accounts
  and external counterparties) where GARBATI only has the first side.

### 10. Webhooks / event-notification system

**Verdict: different by design / not yet relevant, exactly as the task file predicted — confirmed by reading GARBATI's actual internal mechanism, not assumed.**

- Modern Treasury: a mature, externally-facing, versioned, documented product feature for external
  integrators — endpoints registered per organization (separate prod/test URLs), 17 named event topics routed
  via `X-Topic`, HMAC-SHA-256 signatures (`X-Signature`, plus a failsafe `X-Internal-Signature`), per-event
  `X-Webhook-ID` for logical dedup plus per-attempt `X-Delivery-ID`, a 5-second response window, exponential
  backoff, and automatic endpoint pause after 15 consecutive failures.
  [Webhooks](https://docs.moderntreasury.com/reference/webhooks)
- GARBATI has a real, built, but structurally different mechanism: `Garbati.IdentityApi`'s `OutboxMessage`
  (confirmed directly: `Entities/OutboxMessage.cs`, `Entities/OutboxMessageStatus.cs`,
  `Entities/OutboxMessageTypes.cs`, `Contracts/Outbox/AccountFreezeRequestedPayload.cs`) — a durable,
  retry-tracked work queue used today for exactly one purpose, confirmed in `ARCHITECTURE.md` and the
  `DESIGN.md` entity table: propagating an account-freeze instruction from `IdentityApi` to `OrchestratorApi`
  when a customer is closed/deceased. This is the standard transactional-outbox pattern, real and working,
  but it is **internal service-to-service plumbing with exactly one consumer GARBATI itself owns** — not a
  documented, versioned, externally-subscribable event feed for a third-party integrator, because GARBATI has
  no external integrator or customer-facing Channel built yet (`ARCHITECTURE.md`'s own build order: "Channel/
  Gateway... Later tranche," not designed). This is the one concept in this entire brief where the honest
  verdict is "not yet relevant," not "missing" and not "different by design" in the sense of serving a
  different need — GARBATI simply has no external audience yet for a webhook system to serve, exactly the
  framing the task file itself suggested before any of this was checked.

### 11. API idempotency conventions

**Verdict: GARBATI has a real, confirmed, and arguably stricter equivalent — the task file's own prediction, confirmed.**

- Modern Treasury: an `Idempotency-Key` header (max 180 characters) on POST/PATCH requests; a successful
  response is cached and replayed for 24 hours; **keys are scoped per API credential**, not per request route
  — reusing the same key against a different endpoint or a different body within that window returns the
  *original* cached response regardless, not an error; a concurrent request using the same key returns 409.
  [Idempotent Requests](https://docs.moderntreasury.com/platform/reference/idempotent-requests)
- GARBATI's equivalent, confirmed across every service that implements it (`LedgerApi` decisions #42/#44,
  `IdentityApi` decision #29, `OrchestratorApi`'s own `Validation/IdempotencyKeyRule.cs`), is a client-supplied
  idempotency key **plus a stored SHA-256 hash of the request's business fields**, scoped per calling client.
  A retry with the same key and a matching hash returns the original result; **the same key reused with a
  different body is explicitly detected and rejected with 409**, confirmed in `ARCHITECTURE.md` Section 5
  ("same key with a different body returns 409") and `LedgerApi` Section 12.6 ("Same key, different list of
  legs... 409").
- **Worth naming plainly, as the task file asked**: this is one of the few areas where GARBATI's actual
  mechanism is arguably *more defensive* than Modern Treasury's own documented behavior. Modern Treasury's
  own docs describe silently replaying the original cached response when a key is reused against a different
  payload — the caller's new, different request is simply ignored, not flagged. GARBATI's request-hash check
  means a caller that accidentally reuses a key with genuinely different data gets a loud 409, not a silent
  stale replay. This is a real, structural strength of GARBATI's own design, confirmed directly against
  Modern Treasury's own primary documentation, not merely assumed because GARBATI built it first.

### 12. API versioning

**Verdict: could not fully compare — GARBATI's own convention is real and documented; Modern Treasury's current public versioning mechanism could not be confirmed from a primary page read directly in this research.**

- GARBATI: `/v1/` in every service's URL path; a breaking change creates `/v2/` (`ARCHITECTURE.md` Section 5)
  — documented and real, though, as far as this research could tell, not yet exercised in practice (no
  service has shipped a `/v2/` yet).
- Modern Treasury: this research could not locate and directly read a primary Modern Treasury page
  describing their own API's URL-path or header-based versioning scheme (a search surfaced only client-SDK
  semantic-versioning changelogs — a different thing, about library releases, not the API's own request-path
  versioning). **Stated plainly as not verified**, rather than assumed from general knowledge of similar
  API-vendor conventions, per this project's own sourcing discipline.

### 13. Reconciliation (feature-level, not re-researching T-054's own findings)

**Verdict: reinforces T-054, does not change it.**

- Modern Treasury's documentation describes a dedicated Reconciliation guide section (rules, matching, manual
  handling) alongside Expected Payments' own reconciliation-status states (`tentatively_reconciled`/
  `reconciled`/`partially_reconciled`/`unreconciled`) and Returns'/Reversals' own `reconciliation_status`
  field (`unreconciled`/`tentatively_reconciled`/`reconciled`) — this is the feature-level elaboration behind
  the single page (Balances Used in Account Reconciliation) T-054 already cited and read directly.
- This does not change T-054's own conclusion: GARBATI's three named reconciliation gaps (fee-vs-rule check,
  orchestrator-crash sweep) remain closer to internal, exact-keyed consistency checks than to this, because
  GARBATI still has no external bank statement, BAI2 file, or counterparty record to reconcile against. What
  this brief adds, specifically, is Section 6 above (Expected Payments) — the concrete mechanism Modern
  Treasury's own reconciliation feature actually runs on, which T-054 did not need to name because it was
  answering a narrower question (what GARBATI's three *existing* gaps actually are, not what a future
  external-facing reconciliation feature would eventually need to look like).

### 14. Noted but not researched in depth: Account Groups, Automatic Sweeps, FX, Payment Templates, bulk/hosted counterparty onboarding

These appeared in Modern Treasury's real site structure (Section "Site structure actually found" above) but
were not read in depth for this brief, consistent with the task's own focus list and this brief's length
constraints. A brief, honest characterization rather than a silent skip: all of these are features for a
**payments-operations team** managing many counterparties/accounts/currencies on top of a bank relationship
Modern Treasury itself does not own — Account Groups and Sweeps move cash *between* a company's own multiple
accounts; FX converts between currencies for cross-border payments; Payment Templates and bulk/hosted
onboarding speed up high-volume counterparty creation. GARBATI's current single-currency-per-account,
single-institution model, and its complete absence of a counterparty concept (Section 4), make all of these
**likely different-by-design or not-yet-relevant** on the same reasoning as Sections 4–6 and 10 — but this is
a reasoned inference from this brief's other findings, not independently confirmed by reading each of these
specific pages directly, and is flagged honestly as the shallower-confidence item in this brief.

---

## Summary table

| Modern Treasury concept | Verdict | One-line reason |
|---|---|---|
| Ledger / Ledger Account / Ledger Transaction / Ledger Entry | **Equivalent, confirmed built** | GARBATI's `Account`/`Transaction`/`LedgerEntry` match the core shape; no pending/posted/available tiers because GARBATI has no external rail yet to be "pending" against |
| Ledgerable associations | **Different by design** | P-5 (no shared DB/cross-service FK) and decision #34 (LedgerApi calls nobody) structurally rule out a stored polymorphic link; GARBATI correlates by idempotency key/correlation id instead — with a real, minor query-convenience cost |
| Payment Orders (money-movement part) | **Equivalent, but much thinner** | `TransferService` is synchronous, intra-platform, no persisted entity — GARBATI has no external rail to need a richer async lifecycle for yet |
| Payment Orders (multi-step/resumable-state part) | **Equivalent, confirmed built** | `AccountOpening`'s step/status machine is a real comparable "operation with durable state," applied to a different use case |
| Payment Orders (scheduling/external rail) | **Missing, already a known roadmap item** | `ScheduleApi`, Channel/Gateway, real NIBSS — already named "later tranche" in `ARCHITECTURE.md`; not a new finding |
| Counterparties | **Missing, worth considering** | Zero matches anywhere in GARBATI's code; every transfer today must already be GARBATI-internal |
| Virtual Accounts | **Different by design** | Solves "I don't have my own ledger/bank account" — GARBATI's customers already have real `LedgerApi` accounts directly |
| Expected Payments | **Missing, worth considering — most directly relevant to future reconciliation work** | No concept of a declared, awaited external incoming payment; names the concrete mechanism T-054 Q5 left abstract |
| Returns | **Missing, correctly not-yet-relevant** | A Return is a rail telling you your payment was rejected; GARBATI has no external rail to send to yet |
| Reversals | **Equivalent, confirmed built end-to-end** | IdentityApi Section 8 + OrchestratorApi `ReversalService` + LedgerApi chain-reversal, read directly across all three services |
| Approval Rules | **Equivalent, confirmed built in two places** | IdentityApi joint-account M-of-N (customer-side) + OpsApi maker-checker entities (staff-side, confirmed as real code) |
| Legal Entities | **Equivalent for GARBATI's own customers; missing the counterparty-side half** | Same underlying gap as Counterparties, not a separate one |
| Webhooks/events | **Different by design / not yet relevant** | GARBATI's `OutboxMessage` is real but purely internal (IdentityApi→Orchestrator); no external integrator exists yet to serve |
| Idempotency | **Equivalent, confirmed — arguably stricter** | GARBATI's request-hash + 409-on-mismatch is more defensive than Modern Treasury's own documented silent-replay-on-mismatch behavior |
| API versioning | **Could not fully compare** | GARBATI's `/v1/`→`/v2/` convention is real and documented; Modern Treasury's own current scheme could not be confirmed from a primary page read directly |
| Reconciliation (feature-level) | **Reinforces T-054, unchanged conclusion** | Adds Expected Payments as the concrete mechanism; does not alter T-054's "not nostro/vostro yet" finding |
| Account Groups / Sweeps / FX / Templates / bulk onboarding | **Likely different-by-design/not-yet-relevant (lower confidence)** | Reasoned inference from other findings, not independently read page-by-page |

---

## What this does not settle

- **Does not propose building any missing piece found above** (Counterparties, Expected Payments, Returns,
  Legal-Entity-for-counterparties, external webhooks), per the task's explicit scope. These are named as
  honest findings for the owner's own future judgment, not recommendations.
- **Does not re-research reconciliation from first principles** — Section 13 explicitly builds on, and
  defers to, T-054's own Q1–Q5 findings.
- **Is not a referendum on whether GARBATI should become more like Modern Treasury.** Several of the clearest
  "missing" findings here (Counterparties, Virtual Accounts) are missing specifically because GARBATI is a
  different kind of product serving a different role — this brief treats that as the correct, final answer
  for those two, not an open question.
- **Does not independently verify every Modern Treasury claim against Modern Treasury's own production API
  behavior** — this research read Modern Treasury's own public documentation pages directly (not a sandbox
  account, not their OpenAPI spec, not their source code, which is proprietary and not available for a
  "verify, don't trust" code read the way GARBATI's own services were). Where a documentation page's own
  wording was ambiguous or thin (Section 1's `Ledger` top-level container, Section 12's API-versioning
  scheme), this is flagged plainly above rather than filled in by assumption.
- **Does not independently verify `OpsApi`'s full maker-checker workflow logic** — Section 8 confirms the
  relevant entities (`OpsReversalRequest`, `OpsCustomerEditRequest` and their status types) genuinely exist as
  built code, but this research did not read the full service/controller logic behind them, which was outside
  this brief's scope.

## What this affects

This directly informs `ARCHITECTURE.md` Section 9's existing open questions on reconciliation (the token-gap
and orchestrator-crash entries, already citing T-054) by adding one concrete fact T-054 did not have: Modern
Treasury's own named, shaped mechanism (Expected Payments) for the kind of fuzzy, tolerance-windowed matching
T-054 Q5 described only in the abstract as a future need once real NIBSS settlement exists. It also surfaces,
independently of any existing open question, two new honest findings for the owner's own future judgment
(not proposed as tasks): GARBATI's complete absence of a Counterparty/external-payee concept (Section 4,
echoed in Section 9's Legal Entity finding), and the real, P-5-grounded cost of GARBATI's "no ledgerable
association" choice (Section 2) for any future "show me everything about this operation" tooling. Neither is
urgent, both are tied to the already-acknowledged fact (T-054) that GARBATI has no live external-rail
integration today.

## Sources

- [Modern Treasury — Payment Order Object](https://docs.moderntreasury.com/platform/reference/payment-order-object) (fetched and read directly)
- [Modern Treasury — Counterparty Object](https://docs.moderntreasury.com/platform/reference/counterparty-object) (fetched and read directly)
- [Modern Treasury — Virtual Account Object](https://docs.moderntreasury.com/platform/reference/virtual-account-object) (fetched and read directly)
- [Modern Treasury — Expected Payments](https://docs.moderntreasury.com/platform/reference/expected-payments) (fetched and read directly)
- [Modern Treasury — Ledger Account Object](https://docs.moderntreasury.com/platform/reference/ledger-account-object) (fetched and read directly)
- [Modern Treasury — Ledger Transaction Object](https://docs.moderntreasury.com/platform/reference/ledger-transaction-object) (fetched and read directly)
- [Modern Treasury — Ledger Entry Object](https://docs.moderntreasury.com/platform/reference/ledger-entry-object) (fetched and read directly)
- [Modern Treasury — Return Object](https://docs.moderntreasury.com/platform/reference/return-object) (fetched and read directly)
- [Modern Treasury — Reversals](https://docs.moderntreasury.com/platform/reference/reversals) (fetched and read directly)
- [Modern Treasury — Approval Rules Overview](https://docs.moderntreasury.com/payments/docs/approval-rules-overview) (fetched and read directly)
- [Modern Treasury — Legal Entities](https://docs.moderntreasury.com/platform/reference/legal-entities) (found via search, content read via search-summary, not independently re-fetched in full — see caveat below)
- [Modern Treasury — Webhooks](https://docs.moderntreasury.com/reference/webhooks) (fetched and read directly)
- [Modern Treasury — Idempotent Requests](https://docs.moderntreasury.com/platform/reference/idempotent-requests) (fetched and read directly)
- [Modern Treasury — root docs page](https://docs.moderntreasury.com/) (fetched directly; redirected to the Payments product's own guide tree — see "Site structure actually found")
- Internal: `ARCHITECTURE.md` (read directly, Sections 1–12, especially the service map, dependency rules, decisions log, Section 9 open questions)
- Internal: `LedgerApi` source, read directly — `Entities/Account.cs`, `Entities/Transaction.cs`,
  `Entities/LedgerEntry.cs`, `Entities/TransactionChain.cs`, `Entities/LegPurpose.cs`,
  `Entities/EntryType.cs`, `Entities/TransactionStatus.cs`, `Entities/AccountClass.cs`,
  `Entities/AccountStatus.cs`, `Services/Implementations/AccountService.cs`
- Internal: `Garbati.OrchestratorApi` source, read directly — `Entities/AccountOpening.cs`,
  `Entities/AccountOpeningStep.cs`, `Entities/AccountOpeningStatus.cs`,
  `Services/Implementations/TransferService.cs`, `Services/Implementations/ReversalService.cs`
- Internal: `Garbati.IdentityApi/docs/DESIGN.md` Section 8 (decisions #30–#35, reversal authority) and
  Section 9 (decisions log #19–#36), read directly
- Internal: `Garbati.OpsApi` source, confirmed directly (entity files only) —
  `Entities/OpsReversalRequest.cs`, `Entities/OpsReversalRequestStatus.cs`,
  `Entities/OpsCustomerEditRequest.cs`, `Entities/OpsCustomerEditRequestStatus.cs`
- Internal: `research/reconciliation-nibss-integration-patterns.md` (T-054), built on throughout, not re-derived
- Internal: code search across all GARBATI repositories for `Counterparty`/`ExternalAccount`/
  `BankAccountNumber`/`RoutingNumber` (zero matches, confirming Section 4)
- Internal: `AGENTS.md` "Verify, don't trust" section — the standard this research held itself to

## What was attempted and blocked or unreadable (stated plainly)

- `https://docs.moderntreasury.com/ledgers/docs/61-ledgers-overview` — **HTTP 404 Not Found**, direct fetch.
  This exact guessed URL does not exist on the current site; the actual Ledgers reference pages (Ledger
  Account/Transaction/Entry objects) were located instead via search and read directly, and are treated as
  sufficient coverage of the same concepts.
- Modern Treasury's own API URL/header versioning scheme (Section 12) — searched but not found on a primary
  page; only client-SDK semantic-versioning changelogs (a different, unrelated thing) were found. Stated as
  unverified rather than assumed.
- `Legal Entities` (Section 9) — reached via a search-result summary rather than a full direct-fetch read the
  way every other primary page in this brief was; the summary is treated as adequate for this brief's
  purposes (confirming the concept and its rough shape) but was not independently re-verified by a second,
  direct fetch of the full page content the way the other eleven Modern Treasury pages cited above were.
- Modern Treasury's "Account Groups," "Automatic Sweeps," FX, Payment Templates, and bulk/hosted
  counterparty-onboarding pages (Section 14) — seen in the site's navigation structure but not fetched or read
  in depth; characterized by reasoned inference from this brief's other findings only, flagged as the
  lower-confidence item in the summary table.
- `OpsApi`'s full maker-checker workflow/controller logic (Section 8) — only the entity files were read to
  confirm the mechanism genuinely exists as built code; the full approval/rejection logic was not read in
  depth, which is out of this brief's intended scope but stated plainly rather than silently assumed.
