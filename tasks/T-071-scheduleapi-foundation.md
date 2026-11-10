# T-071: ScheduleApi foundation (builds the whole v1 design from T-070)

Owner: builder. Builds the entire v1 (minimal) design ratified in `Garbati.ScheduleApi/docs/DESIGN.md`
in one task — project skeleton, both entities, all 7 endpoints, the scheduling `BackgroundService`,
both downstream clients, migrations. Read `docs/DESIGN.md` in full before starting; it is the spec,
not this summary. This task does not reopen any of its decisions (D-33: no auto-retry; single-holder
only; the narrow `LedgerApi` balance exception) — if something in the design doc turns out to be wrong
or ambiguous once you're in the real code, stop and flag it rather than silently deciding.

## Project shape

Mirror `Garbati.OrchestratorApi`'s scaffold, not `Garbati.StaffIdentityApi`'s or `Garbati.ProductApi`'s
— `ScheduleApi` is the closest match structurally: it's both a server (its own CRUD endpoints) and a
client of two sibling services (`OrchestratorApi` for transfers, `LedgerApi` for the narrow balance
read), the same shape `OrchestratorApi` itself has. Specifically mirror:

- `DownstreamServicesOptions`/`DownstreamServiceOptions` (`Configuration/`) — `BaseUrl`/`ApiKey` per
  downstream service, validated at startup (`IValidateOptions<T>`, `ValidateOnStart`) so a missing
  config value fails loudly at boot, not confusingly inside the first scheduled run. Two entries:
  `OrchestratorApi`, `LedgerApi`.
- The `Clients/Interfaces/` + `Clients/Implementations/` + `Clients/Contracts/` split
  (`IOrchestratorApiClient`/`OrchestratorApiClient`, `ILedgerApiClient`/`LedgerApiClient`), each a thin
  wrapper over `IHttpClientFactory`, each with its own mirrored request/response contract types — do
  not reference the real `Garbati.OrchestratorApi`/`LedgerApi` projects directly, mirror their
  contracts the same way `OrchestratorApi` itself mirrors `LedgerApi`'s.
- API-key authentication for *inbound* requests (the standard pattern every sibling service uses:
  hashed keys in config, `X-Api-Key` header, scope-based `[Authorize(Policy = ...)]`).

## What to build

### 1. Entities and migrations

`StandingInstruction` and `StandingInstructionRun`, exactly as specified in `docs/DESIGN.md` Section 2
— every field, every enum (`AccountType`, `TransferType`, `InstructionType`, `Priority`,
`RecurrenceType`, `RecurrenceFrequency`, `Status` for the instruction; `Status` for the run). Use plain
string contracts for every enum-shaped request/response field (mirroring `LedgerApi`'s own
`Purpose`/`AccountType`/`ChainKind` convention — **do not** declare a request contract field as the raw
C# enum with no `JsonStringEnumConverter`; `LedgerApi` PR #31 shipped exactly this bug and it was only
caught by QA testing the real HTTP endpoint, not by any in-process test — write at least one real
HTTP-level test exercising every enum-shaped request field as the string a real caller would send,
specifically to avoid repeating that exact mistake).

### 2. Endpoints

All 7 from `docs/DESIGN.md` Section 4, under `/v1/standing-instructions`:
- `POST` create — validates and rejects (422, clear reason) any `FromAccountType`/`ToAccountType =
  Loan`, `TransferType != AccountTransfer`, `InstructionType != Fixed`, `RecurrenceType != Periodic`.
- `GET /{id}`, `GET ?customerId=` (paged list)
- `PUT /{id}` — real in-place edit (amount, schedule, priority, `AllowPartialTransfer`); rejected on a
  `Deleted` instruction
- `DELETE /{id}` — soft delete (`Status` → `Deleted`), not a row removal
- `PATCH /{id}/status` — `Active` ↔ `Disabled` only
- `GET /{id}/runs` — paged run history

### 3. The scheduling `BackgroundService`

A plain ASP.NET Core `BackgroundService`, polling hourly (`docs/DESIGN.md` 3.1) — no new NuGet
dependency, no Quartz, no Hangfire. Per tick:

1. Query every `Active` instruction where `ValidFrom <= today <= (ValidTill ?? today)`, today matches
   the recurrence rule (respecting `RecurrenceFrequency`/`RecurrenceInterval`/`RecurrenceOnDay`/
   `RecurrenceOnMonth`, **clamped to the target month's real length** — docs/DESIGN.md 3.4, the
   FINERACT-2914 lesson: a day-31 instruction due in a 30-day month runs on day 30, it does not throw
   and does not silently skip), and `LastRunDate < today`.
2. For each due instruction, compute the idempotency key `sched:{standingInstructionId}:{scheduledDate:yyyy-MM-dd}`
   (docs/DESIGN.md 3.1 — this exact format, confirmed in `ARCHITECTURE.md` 4.3).
3. **If `AllowPartialTransfer` is `true`**: call `ILedgerApiClient`'s balance read for
   `FromAccountId` first (docs/DESIGN.md 3.2). If the real balance is less than `Amount`, submit the
   transfer for the lesser amount (or record `Skipped` with a clear reason if the balance is zero or
   negative — nothing submitted) and record the run as `Partial` with the actual amount. If
   `AllowPartialTransfer` is `false` (the default), skip this balance check entirely — submit the full
   amount and let `OrchestratorApi`'s own rejection produce a `Failed` run on insufficient funds. Keep
   this check narrow: it only ever runs for the opt-in case.
4. Submit to `IOrchestratorApiClient`'s transfer call (`POST /v1/transfers` — mirror the real request
   shape: `IdempotencyKey`, `RequestedBy` = the instruction's `CustomerId`, `DebitAccountId` =
   `FromAccountId`, `CreditAccountId` = `ToAccountId`, `Amount`, `Currency`, `Narration`). Record a
   `StandingInstructionRun`: `Success`/`Partial` with the real `Reference` as `OrchestratorReference`
   on success, `Failed` with the error message on rejection (**no retry** — docs/DESIGN.md 3.3, this is
   the one hard behavioral rule from D-33, test it explicitly: a failed run must not be re-attempted
   before its own next scheduled occurrence).
5. Update `LastRunDate` regardless of outcome (so a `Failed` run doesn't get re-attempted on the next
   tick the same day).

### 4. Authorization scopes

Two new scopes on this service: `scheduleapi.read`, `scheduleapi.write` — matching the table in
`docs/DESIGN.md` Section 4 exactly.

## What this task does not include

- No change to `OrchestratorApi`, `IdentityApi`, `LedgerApi`, `ProductApi`, `OpsApi`, or
  `garbati-ops-console` — both downstream endpoints this service calls already exist with no contract
  change needed (confirmed directly in the design doc). If you find either doesn't actually work the
  way the design doc describes, stop and flag it — don't patch the other repo yourself.
- No financing-module code. Every enum value/field requiring a loan is modeled but its creation path is
  rejected clearly (422) — this task does not make any of them functional.
- No fund-contention priority ordering (docs/DESIGN.md Section 5) — `Priority` is stored, not enforced.
- No staff/ops-console UI.
- No real production API keys for the two downstream clients — use the same dev-key convention every
  sibling service's `appsettings.Development.json` already uses; wiring real production keys (a
  `DownstreamServices__OrchestratorApi__ApiKey`/`DownstreamServices__LedgerApi__ApiKey` on Azure) is
  real infrastructure work for whenever this service actually deploys, same as T-048 was a separate
  task from every other service's own build task.

## Acceptance

- Real test coverage for: each of the 7 endpoints; the create-time rejection of every
  not-yet-supported field combination (Loan account type, non-`AccountTransfer` transfer type,
  `Dues`/`AsPerDues`); the `BackgroundService`'s due-date logic including the day-29-31 clamping case
  specifically (test a day-31 recurrence against a 30-day and a 28/29-day month, confirm it runs on the
  real last day of that month, does not throw, does not skip); the no-auto-retry rule (a `Failed` run
  on one tick is still `Failed`/unretried on the very next tick the same day, only eligible again once
  `LastRunDate` rolls over to the instruction's own next real occurrence); `AllowPartialTransfer`'s
  balance-check path (a real partial-balance scenario produces a `Partial` run with the lesser amount,
  confirmed against a real `ILedgerApiClient` call, not skipped over); the idempotency-key format
  exactly matching `sched:{id}:{date}`; **at least one real HTTP-level test per enum-shaped request
  field**, posting the string value a real caller would send (the `LedgerApi` PR #31 lesson — don't
  let this ship broken the same way and only get caught by QA).
- `docs/DESIGN.md` updated if anything above had to change from what's written there, with the
  deviation and why — same discipline as every other decision log on this platform.
- No production-data migration concern — this is a brand-new service with no existing data anywhere.
