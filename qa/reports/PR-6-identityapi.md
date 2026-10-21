# QA Report — Garbati.IdentityApi PR #6 (T-035)

**Verdict: Approve with nits** (no blockers)

**Post-review fix (coordinator, same PR):** Finding #1 below ("Should fix") was addressed directly on the PR branch — `Email`/`PhoneNumber` now reject a supplied-but-blank value the same way `FullName` already did, with regression tests for both (`Fails_WhenEmailIsSuppliedButBlank`, `Fails_WhenPhoneNumberIsSuppliedButBlank`). 312/312 tests passing (was 308/308, +4). `docs/DESIGN.md` Appendix G updated to describe the fix.

## 1. Scope fidelity — PASS

Diff touches exactly 13 files, 412 insertions / 2 deletions: `docs/DESIGN.md` (doc only), `AuditActions.cs` (one new constant), `UpdateCustomerDetailsRequest.cs` (new), `CustomersController.cs` (one new action), `Program.cs` (one new DI line), `CustomerService.cs`/`ICustomerService.cs` (one new method each), `UpdateCustomerDetailsRequestValidator.cs` (new), plus 5 test files. Confirmed via `git diff main...feat/customer-details-update -- .../CustomerService.cs` that every pre-existing line (`CreateCustomerAsync`, `RecordKycDecisionAsync`, `ChangeStatusAsync`/Suspend/Reactivate/Close/MarkDeceased, `CascadeMandateRevocationAndFreezeAsync`) is byte-for-byte untouched — the new method is a pure insertion. `CustomersController.cs` likewise only gains one new `[Authorize]`-decorated action; no existing route, scope, or response shape changed. No migration (no new columns/tables) — correct, since no new persisted field was needed.

## 2. `CustomerIdentityData`-only, `Customer` untouched — verified by reading the method, not the summary

Read `UpdateDetailsAsync` directly:
```csharp
var identityData = customer.IdentityData!;
if (request.FullName is not null) { ...; identityData.FullName = request.FullName; }
if (request.Email is not null) { ...; identityData.Email = request.Email; }
if (request.PhoneNumber is not null) { ...; identityData.PhoneNumber = request.PhoneNumber; }
identityData.UpdatedAt = DateTimeOffset.UtcNow;
```
Only `identityData.{FullName,Email,PhoneNumber,UpdatedAt}` are ever assigned. `customer.CustomerReference`, `CustomerStatus`, `KycStatus`, `KycTier`, `JurisdictionCode`, and `Customer.UpdatedAt` are never touched anywhere in this method — confirmed, matches the task file's out-of-scope list exactly.

## 3. "Reject if all three null" validation and the empty-string ("") edge case — works, but inconsistent and untested

`UpdateCustomerDetailsRequestValidator.Validate`: all-null → `errors.Add(...)`, 400, service never called (confirmed: `CustomersControllerTests.UpdateCustomerDetails_Returns400_AndDoesNotCallTheService_WhenEveryFieldIsNull` plus `CustomerDetailsHttpTests.UpdateCustomerDetails_Returns400_WhenEveryFieldIsNull` over real HTTP). This part is solid.

The ambiguous case the task didn't spell out — a field present as `""` rather than omitted/`null` — is handled **inconsistently across the three fields**, and untested either way:
- `FullName: ""` → `string.IsNullOrWhiteSpace` catches it → **rejected**, 400 (tested: `Fails_WhenFullNameIsSuppliedButBlank`).
- `Email: ""` → `!string.IsNullOrWhiteSpace(request.Email)` is `false`, so the `@`-check is skipped → validator **passes**. Then in the service, `request.Email is not null` is true (`""` is not null) → `identityData.Email` is overwritten with the literal empty string, and the audit row records `"Email ada@example.com -> "` (blank new value). **No test exercises this.**
- `PhoneNumber: ""` → same story, no format check at all → accepted and silently clears the stored phone number to `""`. **No test exercises this either.**

This mirrors `CreateCustomerRequestValidator`'s own (pre-existing) treatment of `Email`/`PhoneNumber` faithfully, per the task's explicit "don't invent a stricter/looser rule" instruction — so this is not a new regression this PR introduced. But at **create** time there's no "supplied vs. don't-touch" distinction to get confused with; at **update/PATCH** time there is, and the task's own framing ("only supplied fields change") implies a caller sending `""` is "supplying" a value, which here silently becomes "wipe this field to blank" for Email/Phone while being flatly rejected for FullName. That's a real inconsistency a caller (or a reviewer reading an audit row showing a value changing to blank) could trip over, and it's completely unexercised by any test in either direction.

## 4. Audit logging content — verified with a real test, not just "some row exists"

`UpdateDetailsAsync_RecordsAnAuditRow_WithOldAndNewValuesForOnlyTheSuppliedFields` creates a real customer (Email `ada@example.com`), updates only Email, then asserts `audit.Details` literally contains `"Email ada@example.com -> new-email@example.com"` **and** asserts `FullName`/`PhoneNumber` do not appear in `Details` at all. This is a genuine content assertion, not a row-existence check. `EntityType`/`EntityId` are also asserted. Good — matches the task's "old → new values for whichever fields were actually supplied" requirement precisely for the tested case.

## 5. Judgment call #1 (auditing old==new as a "no-op update") — consistent with real `RecordKycDecisionAsync` behaviour, but entirely untested

Read `RecordKycDecisionAsync` directly: it unconditionally writes a `KycVerification` row and an audit row with `Details = "KycStatus {previousStatus} -> {newStatus}; ..."` whenever the customer isn't Closed/Deceased — there is **no** check anywhere that `newStatus != previousStatus` before auditing. So the builder's claim is accurate: `RecordKycDecisionAsync` really does audit a "re-decision" that changes nothing, exactly as described. `UpdateDetailsAsync` follows the same shape (always logs whatever was supplied, regardless of whether it differs from the stored value).

However: **no test in this PR proves this for the new endpoint** — there is no case that creates a customer, re-sends the same `FullName` that's already stored, and asserts the resulting audit row says `"FullName X -> X"` (or that it doesn't create something misleading). Given this is explicitly flagged by the builder as a judgment call, I'd have expected at least one regression test locking in the chosen behaviour, the same way `SuspendCustomerAsync_IsIdempotent_WhenAlreadySuspended` locks in the opposite behaviour for status changes. Low risk in practice (it's a one-line, explainable audit entry, not a money movement), but it's a gap between what was "decided" and what was "tested".

## 6. Scope enforcement — confirmed exact existing policy, no new scope

`[Authorize(Policy = IdentityScopes.CustomerWrite)]` on the new action — `IdentityScopes.CustomerWrite = "identity.customer.write"`, the exact same constant `POST /v1/customers` already uses (confirmed by reading `IdentityScopes.cs`). `AuthorizationTests.cs` gained one new theory row (`PATCH .../details` → `IdentityScopes.CustomerWrite`), exercised through the same 401/403/reaches-endpoint harness every other endpoint on this controller already uses. No new scope constant was added anywhere in `IdentityScopes.cs`.

## 7. DI registration — the fixed bug is actually fixed, and the fix is complete/consistent

`Program.cs` now registers `builder.Services.AddScoped<UpdateCustomerDetailsRequestValidator>();` alongside the other nine validators. Cross-checked: there are exactly 10 validator classes under `Validation/` and exactly 10 matching `AddScoped<...Validator>()` lines in `Program.cs` — one-to-one, nothing else missing. The registration follows the exact same `AddScoped<T>()` pattern as its siblings (no inconsistent lifetime, no factory, no missing interface binding — these validators aren't registered against an interface anywhere in this codebase, consistent with the existing pattern).

## 8. Build and full test suite — ran independently, confirmed

`dotnet build -c Release`: 0 warnings, 0 errors. `dotnet test --no-build -c Release` (Docker available, Testcontainers-backed Postgres, nothing skipped): **308/308 passed**, matching the PR's claim exactly.

## 9. "Does it do exactly what the task says, and nothing else" — confirmed, nothing extra

Endpoint, request contract, validator, service method, audit constant, and test coverage all match the task file's scope list item-for-item. No idempotency key was added (matches the task's explicit "no idempotency key required" instruction). `CustomerResponse` (existing shape) is returned, no new response type. `docs/DESIGN.md` updated (Section 5.1 row + Appendix G), in the same style as prior appendices — no "Proposed" item was built on; this is a plain additive change the task file (D-19, owner-approved) already pre-authorized.

## Findings

1. **[Should fix]** Blank-string (`""`) semantics for `Email`/`PhoneNumber` are inconsistent with `FullName` and untested in either direction (see §3). A caller sending `Email: ""` or `PhoneNumber: ""` passes validation and silently overwrites the stored value with an empty string (not `null`), while the same thing for `FullName` is rejected with 400. Worth either (a) explicitly rejecting blank-but-non-null `Email`/`PhoneNumber` the same way `FullName` is, or (b) deliberately documenting "`""` means clear this field" and adding a test proving it — but right now it's neither decided nor tested, just inherited silently from `CreateCustomerRequestValidator`'s pre-existing gap.
2. **[Nit]** Judgment call #1 (auditing a no-op "resupply the same value" update) is accurately described and consistent with `RecordKycDecisionAsync`'s real behavior, but has no regression test locking it in, unlike the equivalent decision on the status-change side (`SuspendCustomerAsync_IsIdempotent_WhenAlreadySuspended`). Low risk, but worth a one-line test for the next person who touches this code.
3. **[Nit]** `CustomerDetailsHttpTests` deliberately asserts status codes only, not response content, matching the existing `[Collection("Api")]` convention (shared, unreset Postgres) — this is consistent with how other HTTP-level files in this suite already behave, not a new gap introduced by this PR.
4. No money-safety concern: this endpoint never touches balances, mandates, or ledger state; the only "transaction" concern is that the `CustomerIdentityData` update and its audit row are staged on the same `DbContext` and committed by one `SaveChangesAsync` call — confirmed directly in `UpdateDetailsAsync`.
5. No security/secrets concern: audit `Details` records plaintext old/new Email/PhoneNumber values (not BVN/NIN, which doesn't exist on this entity), which is explicitly sanctioned by the task file and consistent with how `RecordKycDecisionAsync`/status-change audits already record plaintext reasons. `AuditLogger.Record` only stages a DB row — no separate `ILogger`/stdout write of `Details`, so no extra PII-logging surface was introduced.

No Blockers.

## What QA could not check

- No real `OpsApi` (T-034) caller exists yet to exercise this endpoint end-to-end from an actual maker-checker workflow — matches the builder's own disclosure.
- Did not exercise the `""`-value edge case against a live HTTP server (traced statically through the validator and service code, which is deterministic and unambiguous, rather than spinning up a server for it) — recommend the owner or a follow-up PR add a test either way.
- Did not review `OpsApi`'s T-034 side (different repo, being built in parallel) for whether it will actually send `""` for an unchanged field — if it does, finding #1 above becomes live rather than theoretical.

## Interview Q&A

1. **If I PATCH `{"email": ""}` on an existing customer, what happens?** It passes validation (the `@`-check is skipped for blank values) and overwrites the stored email with an empty string — unlike `{"fullName": ""}`, which is rejected with 400. This is inconsistent and currently untested either way.
2. **Does `RecordKycDecisionAsync` really audit a decision that changes nothing, or was that just claimed?** Verified directly — it has no equality check between the previous and new `KycStatus`/`KycTier` before writing the audit row; it always audits whatever decision it's handed. `UpdateDetailsAsync` follows the same precedent faithfully.
3. **Does this endpoint ever touch `CustomerStatus`, `KycStatus`, `KycTier`, or `CustomerReference`?** No — read directly, `UpdateDetailsAsync` only ever assigns `identityData.{FullName,Email,PhoneNumber,UpdatedAt}`; nothing on the `Customer` entity is assigned anywhere in that method.
4. **Was the DI bug the builder mentions actually fixed, and is anything else missing?** Yes — `Program.cs` now has exactly 10 `AddScoped<...Validator>()` lines matching the 10 validator classes in `Validation/`, one-to-one; nothing else is missing.
5. **Could a caller use this endpoint to "un-suspend" someone or bypass the maker-checker workflow?** No — it has no interaction with `CustomerStatus` at all (by design, so a suspended customer's typo can still be fixed), and it requires the same `identity.customer.write` scope as every other write on this controller; `OpsApi`'s staging/approval workflow (T-034) isn't built yet and isn't what decides *whether* this endpoint can be called, only *who* calls it once built.
