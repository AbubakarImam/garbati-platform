# T-064: LedgerApi real-payload gaps build (decisions #49-#52, T-063)

Owner: builder. Implements the design ratified in T-063 and written into `LedgerApi/docs/DESIGN.md`
decisions #49-#52 (`main` @ `f3aa1ff`). Read those four decisions in full before starting — they are
the spec. This is LedgerApi's half only. A separate task (T-065) will cover the `Garbati.OrchestratorApi`
side (forwarding `decision_id`, decision #11 in its own `docs/DESIGN.md`) once this lands — don't touch
that repo.

## What to build

1. **Four new nullable columns**, one migration, on both `transactions` and `transaction_chains`:
   - `value_date` (DATE)
   - `branch_code` (string)
   - `channel` (string)
   - `decision_id` (string)

   Follow the exact style of the existing migrations for these tables (e.g. the decision #42/#45
   additive-column migrations) — nullable, no default beyond null, no backfill needed (unlike decision
   #48's chart-of-accounts migration, nothing here has a meaningful value for pre-existing rows).

2. **Request contracts.** Add all four fields, all optional/nullable, to:
   - `Contracts/Requests/TransferRequest.cs`
   - `Contracts/Requests/ChainRequest.cs` (at the **chain level**, not per-leg — these are one value per
     chain/transfer, applying to every leg, matching decision #49-#52's own table notes in `docs/DESIGN.md`
     Section 8)

   Do **not** add them to `ChainLegRequest` or to the reversal request contracts — reversals inherit
   nothing here; a reversed transaction's own `value_date`/`branch_code`/`channel`/`decision_id` stay
   whatever they were (if anything) on the original row, the reversal row gets its own (nullable,
   presumably null unless a caller supplies one for the reversal itself).

3. **Validation — two different mechanisms for two different kinds of check, don't conflate them:**
   - **`branch_code`/`channel`/`decision_id`**: no validation beyond ordinary request-shape checks
     (e.g. reasonable max length — check what, if anything, similar free-text fields like `Narration`
     already enforce and mirror it). These are optional, accept whatever a caller sends.
   - **`value_date`**: a caller-supplied `value_date` that does not equal the transaction's own actual
     creation date (the server's own `DateOnly` for the moment of posting, not the caller's clock) must
     be rejected with **422**, not 400. This is a business-logic rejection, not a request-shape error —
     follow the existing pattern of `InsufficientFundsException`/`CurrencyMismatchException`/etc. in
     `ExceptionHandlingMiddleware.cs` (new exception type, e.g. `ValueDateMismatchException`, mapped to
     422 with a clear message), thrown from the service layer (`TransferService`/`ChainService`) after
     the account/balance checks already run, not from `TransferRequestValidator`/`ChainRequestValidator`
     (those stay 400-only, request-shape checks). A caller that omits `value_date` entirely is fine —
     the row's `value_date` is set to the real creation date either way.

4. **Request-hash scope — do not include any of the four new fields.** Decision #44 already settled
   this exact question for `Narration` ("a description, not part of how money moves... an orchestrator
   that marks retries in the narration would have got 409 for a transfer that succeeded"). The same
   reasoning applies directly to all four new fields: none of them changes which accounts money moves
   between or how much moves. A retry that supplies a different `channel`/`branch_code`/`value_date`/
   `decision_id` for the same idempotency key must still replay the original transaction, not 409.
   Confirm this with a test (same idempotency key, same debit/credit/amount, a *different* value in one
   of the four new fields on retry → 200 replay of the original, not 409) — this is the one easiest
   thing to get backwards by instinct (adding a new field often tempts someone to also add it to the
   hash "for completeness"), so a direct regression test matters here specifically.

5. **Responses.** Add the four fields (as stored — `value_date` always populated after posting, the
   other three only if a caller supplied them) to whatever response shape(s) already return a
   transaction/chain's details (`GET` endpoints, the response from `POST /api/transfers`/`POST
   /api/chains`). Read the real response contracts before editing — don't guess the shape.

## What this task does not include

- No change to `Garbati.OrchestratorApi` or any other service — T-065 covers the one piece that needs
  a change there (forwarding `decision_id`). The other three fields (`value_date`/`branch_code`/
  `channel`) have no caller supplying them yet, by design (decisions #49-#51 explicitly describe this
  as "shape now, content later" — do not invent a caller or wire anything into `OrchestratorApi`
  yourself).
- No change to `ChainService`'s core posting logic, `LegPoster`, `LegReverser`, account locking, or
  balance derivation (decisions #1/#7/#34/#45) — this is purely additive metadata on the envelope, not
  a change to how money moves.
- No deep validation on `branch_code`/`channel` content (no enum, no registry check) — both are
  free-text per decisions #50/#51's own explicit scope.

## Acceptance

- Existing test suite still green, unchanged pass count plus new tests for: the migration (confirm all
  four columns exist, nullable, on both tables), the value-date 422 rejection (a mismatched date fails
  with the right status and message; an omitted date defaults to the real creation date; a matching
  date succeeds), the three free-text fields round-tripping through create → response correctly, and
  — the one that matters most — **the idempotency-hash regression test described in item 4 above**.
- `DESIGN.md` decisions #49-#52 updated if anything above had to change from what's written there, with
  the deviation and why, same discipline as every other decision in that log.
- No production-data migration concern here (every new column is nullable with no backfill) — no
  owner sign-off gate needed before merge, unlike T-061's chart-of-accounts migration.
