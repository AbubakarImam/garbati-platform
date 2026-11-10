# Fineract's real standing-instructions feature set, in depth (deepening T-059 Area 7)

**Task:** No T-number has been assigned yet — the coordinator (per decision #D-5, "only the coordinator and
owner assign task/decision numbers") has not filed a task card for this piece of work; it was commissioned
directly, ahead of `ScheduleApi`'s own design, to deepen one specific finding T-059 already made at a
high level. **Status:** Research only — no design, no schema, no code, no task file, no Sharia ruling, no
interest rate, no profit-sharing ratio.

**What this builds on, and does not re-derive:** `research/fineract-core-functionality-gap-analysis.md`
(T-059, Area 7) already established, at a high level and directly from Fineract's own legacy API
documentation, that Fineract's standing-instruction model is a real, shipped **two-part split** — a
`standinginstructions` definition resource, separate from a `standinginstructionrunhistory`
execution-tracking resource — and that `ScheduleApi`'s own one-line scope statement in `ARCHITECTURE.md`
("recurring instructions and their run history") already independently matches that shape. That fact is
treated here as settled and is not re-argued. This brief goes much deeper: actual field names (confirmed
against Fineract's real Java source and its own generated OpenAPI-derived SDK, not guessed), the real
enum values behind every coded field, the real database columns (fetched directly from Fineract's own
database-table documentation — the same highest-confidence source type `research/real-corebanking-payload-gap-analysis.md`,
T-058, already established as this project's own convention for that site), the real scheduler mechanism,
and several real, named bugs/fixes in Fineract's own issue tracker that double as concrete operational
lessons for whoever eventually designs `ScheduleApi`.

---

## Plain-language summary (read this first)

Fineract's standing instructions are **much more than "move X naira from account A to account B on a
schedule."** The real, shipped feature, confirmed directly from Fineract's own Java source and its own
database schema, is a small combinatorial system: **what kind of transfer** (five real, named types —
ordinary account transfer, loan repayment, charge payment, interest transfer, loan down-payment) crossed
with **what kind of account on each side** (loan or savings, independently on the "from" side and the "to"
side — the database literally has four separate nullable foreign-key columns, one per combination) crossed
with **how the amount for each run is decided** (a fixed configured amount, or "whatever is currently due,"
which can mean a loan's own installment schedule) crossed with **when it runs** (a fixed calendar
cadence, or tied to whatever a linked loan's own due dates are). A standing instruction is not its own
account-transfer mechanism — it is a thin, recurring, scheduled wrapper around the *same* underlying
`AccountTransferDetails` object and `AccountTransferType` enum that an ordinary, one-off, human-initiated
account transfer also uses, confirmed directly from the real database schema (`m_account_transfer_details`
is shared by "account transfer transactions, standing instructions, and loan top-up operations").

Editing is real and supported in place — not just create/cancel — confirmed directly from the real domain
entity's own `update(JsonCommand)` method, which can change the amount, the schedule, or the priority,
with one hard rule: a standing instruction that has already been soft-deleted cannot be edited. "Delete" is
itself a soft operation — the real code flips a `status` column to `DELETED` and renames the record (to
free its unique name for reuse), it does not remove the row.

The run-history resource is real, lean, and specific: one row per execution attempt, holding the
instruction it belongs to, a status (confirmed as a plain string column, not a shared numeric code), the
actual amount that moved (which matters because Fineract supports **partial transfers** — a real, shipped,
opt-in feature, `allowPartialTransfer`, that lets a standing instruction take whatever the source account
can actually afford rather than failing outright when the full amount isn't available), a timestamp, and a
nullable error message. Failure handling has a genuinely interesting, confirmed history: Fineract used to
have a real bug where one failed instruction in a batch of many rolled back *everyone's* successful
transfers for that run; this was fixed so each instruction now executes in its own isolated database
transaction, and a failure is recorded, logged, and left alone until the instruction's own next scheduled
occurrence — Fineract does **not** auto-retry a failed standing instruction later the same day or
immediately, which is the single most load-bearing fact for `ARCHITECTURE.md` Section 9's own still-open
P-3 question.

The scheduling mechanism is real and named: a job called **Execute Standing Instruction(s)**, implemented
as a Spring-Batch-style "tasklet" class (`ExecuteStandingInstructionsTasklet`), running on Fineract's
shared Quartz-based job scheduler — the same scheduler infrastructure underlying every other periodic job
in Fineract (including the interest-posting jobs T-059's Area 9 already found). It was later re-architected
to run as a **partitioned** batch job, split by source account, specifically to stop one giant transaction
from serializing every instruction in the system. A separate, general (not standing-instruction-specific)
scheduler setting controls whether an entirely missed job run — because the server itself was down at the
scheduled time — gets replayed afterward, and it is **off by default**.

One genuinely useful, concrete, real-world lesson surfaced along the way, worth flagging explicitly for
whoever designs `ScheduleApi`'s own monthly-recurrence logic: Fineract shipped a real, later-fixed bug
(FINERACT-2914) where a standing instruction recurring "on the 31st" would either crash the entire job
outright in short months, or silently skip execution in every month shorter than 31 days, because the
original code used naive date-truncating arithmetic instead of clamping the configured day to each month's
actual length. This is not a hypothetical edge case to anticipate — it is a bug Fineract's own users hit in
production.

One priority/ordering question is left genuinely open below, not papered over: a numeric/named priority
field (`URGENT`/`HIGH`/`MEDIUM`/`LOW`) definitely exists on every standing instruction, but whether — and in
which direction — the execution job actually uses it to decide which instruction wins when several draw on
the same underrun account on the same day could not be confirmed from a solid primary source; the one
source found making that claim was ambiguous about sort direction and partly traced to an unofficial
third-party fork rather than Fineract's own current `develop` branch.

---

## Findings

### 1. Types of standing instructions — far more than plain account-to-account transfers

**Confidence: high.** Confirmed directly from Fineract's own Java enum source files, fetched and read
verbatim (not summarized), and cross-checked against Fineract's own database-documentation site.

A standing instruction's **`transferType`** field is the shared `AccountTransferType` enum — the *same*
enum an ordinary, one-off, human-initiated account transfer uses, not a standing-instruction-specific type.
Fetched directly from
[`AccountTransferType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/AccountTransferType.java),
verbatim:

```java
INVALID(0, "accountTransferType.invalid"),
ACCOUNT_TRANSFER(1, "accountTransferType.account.transfer"),
LOAN_REPAYMENT(2, "accountTransferType.loan.repayment"),
CHARGE_PAYMENT(3, "accountTransferType.charge.payment"),
INTEREST_TRANSFER(4, "accountTransferType.interest.transfer"),
LOAN_DOWN_PAYMENT(5, "accountTransferType.loan.downpayment");
```

Separately, **which kind of account** sits on each side is its own independent field (`fromAccountType`/
`toAccountType`), the shared `PortfolioAccountType` enum, fetched directly from
[`PortfolioAccountType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/PortfolioAccountType.java):

```java
INVALID(0, "accountType.invalid"),
LOAN(1, "accountType.loan"),
SAVINGS(2, "accountType.savings"),
```

This is confirmed at the **schema level, not just the enum level**: Fineract's own database documentation
for the shared `m_account_transfer_details` table (fetched directly,
[fineract.apache.org/docs/database/tables/m_account_transfer_details.html](https://fineract.apache.org/docs/database/tables/m_account_transfer_details.html))
lists **four separate, independently-nullable** foreign-key columns — `from_savings_account_id`,
`to_savings_account_id`, `from_loan_account_id`, `to_loan_account_id` — plus `from_office_id`/`to_office_id`/
`from_client_id`/`to_client_id`/`transfer_type`, with the page's own description stating plainly this table
"serves as a reference point for account transfer transactions, standing instructions, and loan top-up
operations." In other words: a standing instruction does not have its own private notion of "account" —
it points at the same shared account-transfer-details object an ordinary transfer uses, and that object can
legitimately link a savings account to a loan account, a loan account to a savings account, or either to
itself in kind.

A third, independent axis — **how the per-run amount is decided** — is the `instructionType` field, the
`StandingInstructionType` enum, fetched directly from
[`StandingInstructionType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/StandingInstructionType.java):
only two real values, `FIXED(1)` (the same configured `amount` every run) and `DUES(2)` (whatever amount
is currently owed — e.g. a loan's own next installment). **This corrects a secondary source** (see "What
was attempted" below): a Finecko guide fetched during this research claimed a third `instructionType` value
("Outstanding balance," id 3) that does not exist in Fineract's real source; the real enum has exactly two
non-`INVALID` values, not three.

**Net answer to the question asked:** no, it is not only account-to-account transfers. Fineract's real,
shipped standing-instruction model explicitly supports savings-to-savings transfers, savings-to-loan
auto-debit repayment, loan-related charge payment, interest transfer, and loan down-payment, crossed with
either side independently being a loan or a savings account, crossed with the amount being either fixed or
"whatever is due" — a genuinely richer combinatorial space than a single transfer-type enum would suggest.

### 2. Recurrence/frequency options — periodic or dues-linked; an end date or no end date; no occurrence count

**Confidence: high** on the real fields and their absence of a count-limit (confirmed directly from
Fineract's own database-table documentation); **medium** on the exact semantics of `recurrenceOnDay`/
`recurrenceOnMonth` per frequency type (not independently confirmed from source, flagged below).

The **recurrence category** itself is `recurrenceType`, the `AccountTransferRecurrenceType` enum, fetched
directly from
[`AccountTransferRecurrenceType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/AccountTransferRecurrenceType.java):

```java
INVALID(0, "accountTransferRecurrenceType.invalid"),
PERIODIC(1, "accountTransferRecurrenceType.periodic"),
AS_PER_DUES(2, "accountTransferRecurrenceType.as.per.dues"),
```

`PERIODIC` runs on a fixed calendar cadence (`recurrenceFrequency`/`recurrenceInterval`, below); `AS_PER_DUES`
fires whenever something is actually owed (tying back to `instructionType = DUES`) rather than on an
independent calendar — e.g. a loan's own real repayment schedule, which is itself the mechanism a real,
merged Fineract PR titled "Fix standing instruction not created at loan disbursement" ([apache/fineract
PR #6235](https://github.com/apache/fineract/pull/6235), title/existence via search-engine summary, not
independently fetched in full) auto-creates a standing instruction for at loan disbursement time.

For `PERIODIC` recurrence, Fineract's own database documentation for the standing-instruction definition
table — fetched directly,
[fineract.apache.org/docs/database/tables/m_account_transfer_standing_instructions.html](https://fineract.apache.org/docs/database/tables/m_account_transfer_standing_instructions.html)
— confirms these columns exist, verbatim column names: `recurrence_type` (TINYINT, not nullable),
`recurrence_frequency` (SMALLINT, nullable), `recurrence_interval` (SMALLINT, nullable), `recurrence_on_day`
(SMALLINT, nullable), `recurrence_on_month` (SMALLINT, nullable), `valid_from` (DATE, not nullable),
`valid_till` (DATE, **nullable**), `last_run_date` (DATE, nullable).

**What this confirms about the specific sub-questions asked:**
- **Daily/weekly/monthly/yearly as real, shipped options:** yes — `recurrence_frequency` is a small integer
  code consistent with Fineract's general days/weeks/months/years frequency vocabulary used elsewhere in
  the platform (its shared `PeriodFrequencyType` enum, fetched directly from
  [`PeriodFrequencyType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/common/domain/PeriodFrequencyType.java),
  reads `DAYS(0)`, `WEEKS(1)`, `MONTHS(2)`, `YEARS(3)`, `WHOLE_TERM(4)`, `INVALID(5)`, `SEMI_MONTHLY(6)`) —
  **but this research could not independently confirm that the standing-instruction `recurrence_frequency`
  column is literally bound to this exact shared enum class** rather than its own narrower one; a separate,
  search-engine-only source (Finecko) claimed a 1-indexed Days/Weeks/Months/Years scheme specific to
  standing instructions, which does not match `PeriodFrequencyType`'s own 0-indexed values — flagged as an
  unresolved discrepancy between a directly-fetched general-purpose enum and a search-summarized,
  possibly-inaccurate standing-instruction-specific claim, not resolved either way here.
- **A specific day-of-month:** yes, `recurrence_on_day`/`recurrence_on_month` are real, separate integer
  columns. **What this research could not confirm:** whether `recurrence_on_day` is reused to mean
  "day of the week" when `recurrence_frequency = WEEKS` (as opposed to always meaning "day of the month"),
  since Fineract exposes only one composite request parameter for this
  (`recurrenceOnMonthDay`, format `"dd MMMM"`, confirmed directly as a real constant name in
  `StandingInstructionApiConstants.java`, below) — the exact per-frequency-type semantics of this single
  pair of columns were not independently verified from the read/write service's own logic.
- **An end date, a count limit, or "until further notice":** confirmed directly — `valid_till` is a real,
  **nullable** `DATE` column, and `valid_from` is required. No occurrence-count/number-of-runs column or
  field was found anywhere in the confirmed schema or the confirmed API constants list. **Net answer:**
  Fineract supports an end date (set `valid_till`) or no end date at all (leave `valid_till` null — "until
  further notice"); it does **not** support "stop after N occurrences" as a distinct, named mechanism.

**A concrete, real operational lesson, not hypothetical:** Fineract shipped and later fixed a real bug,
[FINERACT-2914, "Execute Standing Instruction job fails or skips months when the recurrence day is
29-31"](http://www.mail-archive.com/issues@fineract.apache.org/msg30000.html) (fetched and read directly).
Quoting the bug's own two confirmed failure modes: a standing instruction dated to recur on day 31 **threw
an unhandled date exception and crashed the entire job** in any month shorter than 31 days (e.g. "Invalid
date 'APRIL 31'"), and separately, because the due-date calculation used date-truncating arithmetic rather
than clamping to the actual month length, **the instruction silently never ran at all** in February, April,
June, September or November — executing only 7 times a year instead of 12, with no error logged. This is a
real, named, user-hit edge case directly relevant to any future `ScheduleApi` month-day recurrence design,
not an invented hypothetical.

### 3. Request/response field shape — the user's guessed field list, confirmed and corrected

**Confidence: high.** Confirmed directly from two independent primary-adjacent sources that converge
exactly: Fineract's own real Java API-constants source file, and a TypeScript SDK auto-generated from
Fineract's own OpenAPI/swagger specification.

Fetched directly,
[`StandingInstructionApiConstants.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-account-transfer/src/main/java/org/apache/fineract/portfolio/account/api/StandingInstructionApiConstants.java)
(current location, after the account-transfer module move, FINERACT-3121) — the real, verbatim parameter-name
constants for a standing instruction's own fields: `nameParamName`, `priorityParamName`,
`instructionTypeParamName`, `statusParamName`, `amountParamName`, `validFromParamName`, `validTillParamName`,
`recurrenceTypeParamName`, `recurrenceFrequencyParamName`, `recurrenceIntervalParamName`,
`recurrenceOnMonthDayParamName`, `monthDayFormatParamName`, and — not in the user's guessed list, a real,
separate, shipped field — **`allowPartialTransferParamName`** (see Finding 5).

Separately, the shared from/to-account-level fields (`fromOfficeId`, `fromClientId`, `fromAccountType`,
`fromAccountId`, `toOfficeId`, `toClientId`, `toAccountType`, `toAccountId`, `transferType`) live one layer
down, on the shared `AccountTransferDetails`/`AccountTransfersApiConstants` object every ordinary transfer
also uses — confirmed by the combined response model fetched directly from Fineract's own OpenAPI-generated
SDK,
[`get-page-items-standing-instruction-swagger.ts`](https://app.unpkg.com/fineract_sdk@0.0.5/files/models/get-page-items-standing-instruction-swagger.ts),
quoted verbatim:

```typescript
export interface GetPageItemsStandingInstructionSwagger {
  id?: number;
  accountDetailId?: number;
  name?: string;
  fromOffice?: GetFromOfficeStandingInstructionSwagger;
  fromClient?: GetFromClientStandingInstructionSwagger;
  fromAccountType?: GetFromAccountTypeStandingInstructionSwagger;
  fromAccount?: GetFromAccountStandingInstructionSwagger;
  toOffice?: GetToOfficeStandingInstructionSwagger;
  toClient?: GetToClientStandingInstructionSwagger;
  toAccountType?: GetToAccountTypeStandingInstructionSwagger;
  toAccount?: GetToAccountStandingInstructionSwagger;
  transferType?: GetTransferTypeStandingInstructionSwagger;
  priority?: GetPriorityStandingInstructionSwagger;
  instructionType?: GetInstructionTypeStandingInstructionSwagger;
  status?: GetStatusStandingInstructionSwagger;
  amount?: number;
  validFrom?: string;
  recurrenceType?: GetRecurrenceTypeStandingInstructionSwagger;
  recurrenceFrequency?: GetRecurrenceFrequencyStandingInstructionSwagger;
  recurrenceInterval?: number;
  recurrenceOnMonthDay?: string;
}
```

(This specific generated response model happens not to list `validTill`, `accountDetailId`'s internal shape,
or `allowPartialTransfer` — both are confirmed to be real, request-side fields from the Java constants file
and the domain-entity summary below regardless; this is one generated response DTO, not evidence they don't
exist.)

The real underlying domain entity confirms the full, combined field set and the actual database table it
maps to — fetched directly and summarized faithfully from
[`AccountTransferStandingInstruction.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-account-transfer/src/main/java/org/apache/fineract/portfolio/account/domain/AccountTransferStandingInstruction.java)
(the fetch tool returned a structured summary of the file's real content, not the raw text itself, so this
is graded one notch below a verbatim quote): table `m_account_transfer_standing_instructions`, with a
many-to-one link to `AccountTransferDetails`, `name` (unique), `priority`/`instructionType`/`status` as
integers, `amount` as `BigDecimal(19,6)`, `validFrom`/`validTill` as `LocalDate`, `recurrenceType`/
`recurrenceFrequency`/`recurrenceInterval` as integers, `recurrenceOnDay`/`recurrenceOnMonth` as integers,
`lastRunDate` tracking the most recent execution, and `allowPartialTransfer` as a boolean — independently
matching, column-for-column, the database-table documentation page quoted in Finding 2 above.

**Direct answer to the field list the task asked to confirm or correct:** `fromOfficeId`, `fromClientId`,
`fromAccountId`, `toOfficeId`, `toClientId`, `toAccountId`, `transferType`, `instructionType`, `priority`,
`status`, `amount`, `validFrom`, `validTill`, `recurrenceType`, `recurrenceFrequency`, `recurrenceInterval`,
`recurrenceOnMonthDay` — **all confirmed real**, exactly as guessed. **What the guess was missing**: a
required, unique `name` field; a separate `fromAccountType`/`toAccountType` field (loan vs. savings,
needed alongside the id to know which of the four FK columns it resolves to); `allowPartialTransfer`
(boolean); and the standard Fineract write-request envelope fields (`locale`, `dateFormat`,
`monthDayFormat`) that accompany every Fineract write call, not unique to this resource.

### 4. Status/lifecycle — Active, Disabled, (soft-)Deleted; no persisted "Expired" code found

**Confidence: high** on the three real persisted status codes (confirmed directly from Fineract's own
enum source); **medium** on how "expired" is actually surfaced, since this research found no persisted
fourth status value and could not independently confirm the exact read-time logic that treats a
past-`validTill` instruction as expired.

Fetched directly,
[`StandingInstructionStatus.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/StandingInstructionStatus.java):

```java
INVALID(0, "standingInstructionStatus.invalid")
ACTIVE(1, "standingInstructionStatus.active")
DISABLED(2, "standingInstructionStatus.disabled")
DELETED(3, "standingInstructionStatus.deleted")
```

**This corrects a secondary source directly**: the same Finecko guide that over-counted `instructionType`
(Finding 1) also claimed a fourth, persisted `status = 4` ("Expired," "triggered automatically when
`validTill` passes"). The real enum has no such value. The much more likely real mechanism — not
independently confirmed from the read-service's own query logic, flagged honestly as inferred rather than
verified — is that "expired" is a **derived, read-time label** (today's date is past a still-`ACTIVE`
row's own `validTill`), not a fourth stored lifecycle state the execution job or any API call ever writes.

**Real, confirmed transitions:** create (starts `ACTIVE`); an update call can change `status` to `DISABLED`
(the mechanism for "pause without deleting," confirmed as a real, distinct value, though this research did
not independently find a dedicated "disable" sub-command separate from the general update endpoint — it is
plausibly just a normal field update); and a real `delete()` method on the domain entity (confirmed directly
from the source-code summary) sets `status = DELETED` **and renames the record** by appending a
`"_deleted_{id}"`-style suffix to its own `name` — a genuine **soft delete**, not a row removal, done
specifically so the now-freed unique `name` can be reused by a future standing instruction. This matches
T-059's own prior finding (Fineract's legacy API documentation lists a `DELETE` operation on
`standinginstructions/{id}`) with the added, newly-confirmed detail that this "delete" is implemented as a
status flip plus a rename, not a SQL row deletion.

### 5. The run-history resource — lean, string-status, logs partial transfers too

**Confidence: high** on the schema (fetched directly from Fineract's own database-table documentation,
cross-confirmed by an independent Java-source summary); **medium** on the exact literal status strings used.

Fetched directly,
[fineract.apache.org/docs/database/tables/m_account_transfer_standing_instructions_history.html](https://fineract.apache.org/docs/database/tables/m_account_transfer_standing_instructions_history.html) —
verbatim column list: `id` (BIGINT, not nullable), `standing_instruction_id` (BIGINT, not nullable, FK to the
definition table), `status` (**VARCHAR(20)**, not nullable — a plain string, notably *not* the same small
integer-coded scheme the definition table's own `status` column uses), `execution_time` (DATETIME, not
nullable), `amount` (DECIMAL(19,6), not nullable — the amount actually transferred on that run, which can be
less than the instruction's configured amount), `error_log` (VARCHAR(500), nullable).

**What gets logged per execution, confirmed converging across a direct database-schema fetch, a direct
GitHub Java-source fetch, and a search-engine-summarized real merged PR
([apache/fineract PR #6727, "Use bind parameters for standing instruction history insert"](https://github.com/apache/fineract/pull/6727),
fetched directly, confirming the insert writes exactly `standing_instruction_id`, `status`, `amount`,
`error_log`):** which instruction ran, when, whether it succeeded/partially succeeded/failed (as a literal
string, not an integer code — a real GitHub issue,
["Standing Instructions History: the 'Error Log' column header reads 'Operation failed.'"](http://www.mail-archive.com/issues@fineract.apache.org/msg30512.html)
(title confirmed via search, not independently fetched in full), corroborates that at least a "failed"-style
literal string status exists and is user-visible), how much actually moved, and — only on failure — a free-text
error message capped at 500 characters.

**Failure handling, the single most load-bearing fact for `ARCHITECTURE.md` Section 9's open P-3 question,
confirmed directly from a real, merged Fineract PR's own description:**
[apache/fineract PR #6806, "Fix rollback of all standing instructions when one fails"](https://github.com/apache/fineract/pull/6806)
(fetched and read directly). **The old, buggy behavior:** "the entire step transaction rolled back, undoing
all transfers, history rows, and `last_run_date` updates — even for instructions that had succeeded" whenever
any one instruction in a batch run failed. **The fixed, current behavior:** "each instruction now executes in
its own `REQUIRES_NEW` transaction. Successful or partial transfers commit immediately with their history
row. Failed transfers roll back their transfer attempt but record a separate failed history row that
persists. The job still reports as failed overall, but individual instruction outcomes are isolated." **No
automatic retry of a failed instruction later the same run, or before its own next scheduled occurrence, was
found anywhere** — a failed instruction simply sits as a `FAILED`-style history row with an error message
until its own recurrence rule fires it again naturally. This is a direct, real-world, shipped answer to
P-3's own question ("does a failed run get retried automatically, or just recorded and left for the customer
to notice") — Fineract's real answer is **recorded and left alone, not auto-retried**, matching the
architecture entry's own framing of the traditional, non-retried standing-order model more than the
re-presented-direct-debit model.

A genuinely real, shipped partial-success case also exists: `allowPartialTransfer` (confirmed as a real
field, Finding 3). Its own proposal thread,
[FINERACT-2400, fetched and read directly](http://www.mail-archive.com/dev@fineract.apache.org/msg12501.html),
states the mechanism plainly: when enabled and the required amount exceeds what the source account can
actually release (computed via the savings account's own `getWithdrawableBalance()`, which already accounts
for minimum-balance and overdraft rules), the system "clamp[s] the transfer to the available amount and
log[s] it; skip[s] the instruction when nothing is available," and records the partial transfer as a
**success** with the actual (smaller) amount, not as a failure. A search-engine summary (medium confidence,
not independently re-verified against source) indicates the JIRA ticket behind this was marked **Resolved/
Fixed, shipped in Fineract 1.17.0** — i.e. this is a real, merged feature, not just a proposal.

**What this research could not verify:** the complete, exact set of literal status strings the `status`
VARCHAR column actually stores (e.g. whether it is exactly `"success"`/`"partial"`/`"failed"`, some other
casing, or a localization-key-style string matching the enum pattern used elsewhere) — every source found
agrees on the *concept* (success/partial/failed) but none gave an independently-verified, authoritative
literal string list.

### 6. Priority — the field is real; the ordering mechanism is not confirmed

**Confidence: high** on the field's existence and its real enum values (fetched directly from source);
**low** on whether/how the execution job actually enforces priority ordering when funds are short.

Fetched directly,
[`StandingInstructionPriority.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/StandingInstructionPriority.java):

```java
INVALID(0, "standingInstructionPriority.invalid"),
URGENT(1, "standingInstructionPriority.urgent"),
HIGH(2, "standingInstructionPriority.high"),
MEDIUM(3, "standingInstructionPriority.medium"),
LOW(4, "standingInstructionPriority.low")
```

`priority` is a required field on every standing instruction (Finding 3), and its options are correctly
serialized to the template endpoint's response — confirmed by a real, merged PR,
[apache/fineract PR #6540, "Serialize standing instruction template options"](https://github.com/apache/fineract/pull/6540)
(fetched directly), whose own description names "Priority" as one of the real option lists the
`GET .../standinginstructions/template` endpoint returns (alongside "Type," "Status," "From/To Office,"
"From/To Account Type," "Standing Instruction Type," "Recurrence Type," "Recurrence Frequency" — this PR's
own field list is itself useful independent confirmation that the field names found elsewhere in this
brief are real and match how Fineract's own codebase groups them).

**What could not be confirmed:** whether the execution job actually reads and sorts by this field when
several standing instructions on the same source account are due the same day and the account cannot cover
all of them. The one source making a specific claim here — a search-engine summary describing an
`ORDER BY atsi.priority DESC` clause in the instruction-retrieval query, and separately noting that, "while
the enum is `URGENT(1)..LOW(4)`," a descending sort would process the *lowest*-priority instructions first —
was not independently re-fetched and verified against apache/fineract's own current `develop` branch source,
and part of the surrounding search results pointed at third-party forks (`COG-GTM/fineract-Core-Banking1`,
a `devin-ai-integration[bot]`-authored fork PR) rather than the upstream project itself. **This research
treats the field's existence as confirmed and its operational enforcement — and even its correct sort
direction, if it exists at all in upstream Fineract — as genuinely unresolved, not papered over as a small
detail.** A separate, lower-confidence search-engine composite summary (the same Finecko guide already
flagged for two other inaccuracies above) describes a plausible, simple intended behavior — "lower value =
higher priority, determines execution order when the account balance is insufficient to cover all of them" —
which is directionally consistent with `URGENT=1` being the most urgent value, but this remains a
description of *intent*, not a verified behavior of the shipped execution code.

### 7. Editing an existing standing instruction — yes, real, in-place, with one hard rule

**Confidence: high.** Confirmed directly from the real domain entity's own method, and matching the
already-known `PUT .../standinginstructions/{id}` endpoint from T-059's own prior, directly-fetched finding.

The real domain class's own `update(JsonCommand)` method (fetched directly from source, summarized
faithfully, Finding 3's citation) "modifies fields and returns a Map of changes; validates that deleted
instructions cannot be modified." This is a genuine update-in-place, not a cancel-and-recreate workflow:
amount, schedule (`validFrom`/`validTill`, the recurrence fields), and priority can all be changed on an
existing, active standing instruction via this path, matching the already-documented `PUT` endpoint. The
one confirmed, real constraint: a soft-deleted (`status = DELETED`) instruction's own `update()` call
rejects the change — once deleted, a standing instruction is frozen, not editable back to life.

### 8. The scheduled-job mechanism — a named job, a Quartz-based scheduler, later made partition-scalable

**Confidence: medium-high.** The job's real existence and name are confirmed by three independent,
converging sources (two fetched directly, one via search); the underlying scheduler framework (Quartz) is
confirmed by a direct quote from a real, merged Fineract issue thread; the Spring-Batch-style
"Tasklet"/"partitioned" architecture is inferred from real, specific vocabulary in two directly-fetched PR
descriptions, not from an independently-fetched framework-dependency declaration.

The job is named **"Execute Standing Instruction"** (singular form) or **"Execute Standing Instructions"**
(plural form appears in some sources) — confirmed converging across: (a) a direct fetch of
[apache/fineract PR #6826, "Move account transfers into a fineract-account-transfer module"](https://github.com/apache/fineract/pull/6826),
whose own description states the module move carried "the entities... repositories, services, API
resources, command handlers, and the 'Execute Standing Instruction job'" with it; (b) a direct fetch of
[apache/fineract-consumer-facing issue #239](https://github.com/apache/fineract-consumer-facing/issues/239),
quoting its own stated design intent verbatim: "Fineract Core stays the system of record. Instructions are
Core standing instructions, and Core's *Execute Standing Instruction* job runs them" — i.e. a separate,
newer Fineract-adjacent consumer-facing project deliberately chose to **not** reimplement scheduling, and
instead wrap Core's own existing job; and (c) a search-engine summary independently naming the implementing
class `ExecuteStandingInstructionsTasklet` and its package,
`org.apache.fineract.portfolio.account.jobs.executestandinginstructions` — not independently re-fetched as
raw source in this research, but specific and consistent enough across multiple results to treat as real.

The underlying scheduler framework is **Quartz** (a well-known, real, open-source Java job-scheduling
library) — confirmed by a direct quote from
[FINERACT-2923, "Execute missed scheduler job runs after restart or resume"](http://www.mail-archive.com/issues@fineract.apache.org/msg31164.html)
(fetched and read directly): "A fire while suspended is vetoed, which Quartz counts as executed, not
missed." This same thread confirms a real, general (not standing-instruction-specific) scheduler setting,
`fineract.job.execute-missed-runs` (**default: `false`**), controlling whether a job whose `next_run_time`
fell in the past while the server was down gets replayed after restart — i.e. **by default, Fineract does
not catch up an entirely missed scheduled run**, a separate and distinct fact from the per-instruction
failure-handling behavior already confirmed in Finding 5 (a missed *server-downtime* window is not the same
thing as a *failed-at-execution-time* instruction).

The job was later re-architected for scale: a real, merged PR,
[FINERACT-2672/apache/fineract PR #6078, "Run standing instructions as a partitioned batch job"](https://github.com/apache/fineract/pull/6078)
(existence and title confirmed via search, not independently fetched in full), partitions the work by
source account rather than running every due instruction inside one single transaction — the terms "Tasklet"
and "partitioned... job" found across these sources are standard Spring Batch vocabulary, consistent with
Fineract being a Spring-based Java application generally, though this research did not independently fetch
a `build.gradle`/`pom.xml` dependency declaration naming Spring Batch explicitly, so this specific
framework-identification is graded one notch below the Quartz finding, which has a direct quote naming it
outright.

---

## What this does not settle

- **Does not design `ScheduleApi`, propose a schema, or write any code** — exactly as commissioned, every
  finding above is a report of what Fineract's own real, shipped system does, not a recommendation for what
  GARBATI's own `ScheduleApi` should do.
- **Does not resolve `ARCHITECTURE.md` Section 9's own P-3 question**, though it gives the question's own
  framing a concrete, real, shipped data point: Fineract's real answer to "does a failed standing-order run
  get retried automatically" is **no — it is recorded as a failed history row with an error message and left
  for the instruction's own next natural scheduled occurrence**, not auto-retried sooner. The decision for
  GARBATI's own `ScheduleApi` remains the owner's; this is new input, not a resolution.
- **Does not independently verify the exact literal status strings** the run-history `status` column stores
  (Finding 5), nor **the operational enforcement (or even existence, in current upstream code) of
  priority-based ordering** when several instructions compete for one underfunded account (Finding 6) — both
  named plainly as open rather than guessed at.
- **Does not verify the exact per-frequency-type semantics of `recurrence_on_day`/`recurrence_on_month`**
  (Finding 2) — specifically, whether `recurrence_on_day` means "day of the week" under a weekly frequency
  as opposed to always "day of the month."
- **Does not make, imply, or suggest any Sharia ruling.** One honest, narrow observation worth flagging for
  whoever eventually designs a GARBATI-financing-aware equivalent of Fineract's `instructionType = DUES`/
  `recurrenceType = AS_PER_DUES` mechanism (an amount/schedule that tracks a *linked account's own current
  obligation* rather than a fixed independent number): Fineract's own "amount currently due" concept is built
  for conventional, interest-bearing amortizing loans. Whatever "amount currently due" means for a future
  GARBATI Murabaha/Musharaka/Mudarabah-style financing contract's own installment or profit-distribution
  schedule is a real, separate, contract-type-specific question this research does not attempt to answer,
  and would need a qualified Sharia scholar's input before being built — the same precondition D-28 already
  names for the financing module generally, not a new gate invented here.
- **Does not independently verify Fineract's Spring Batch dependency** by name (Finding 8) — inferred from
  vocabulary, not fetched from a dependency manifest.

## What this affects

This research directly feeds `ARCHITECTURE.md` Section 9's existing **P-3 ("Standing order retry-on-failure")**
entry — the same entry T-059 already touched at a high level. The new, concrete input this brief adds: Fineract's
real, shipped, production behavior (after a real bug fix, PR #6806) is **per-instruction transaction
isolation plus no automatic retry** — a failed run is logged with an error message and left alone until the
instruction's own next scheduled occurrence fires it again naturally, and a separate, general,
opt-in-and-default-off scheduler setting governs only the different question of replaying an *entirely
missed* job run after server downtime. This is now real, sourced, shipped-system evidence for that open
question, not a new question — the decision for `ScheduleApi`'s own eventual retry behavior remains the
owner's.

It also gives `ScheduleApi`'s eventual design concrete, named, real field-level and lifecycle detail to
check against once that design actually starts (the full field list in Finding 3, the real soft-delete
mechanism in Finding 4, the real partial-transfer feature in Finding 5, and the real day-29-31 recurrence
bug in Finding 2) — none of this is proposed for GARBATI to copy; it is offered as the real, concrete shape
of a comparable, shipped system for whoever designs `ScheduleApi` to weigh against.

## Sources

- [`AccountTransferType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/AccountTransferType.java) — fetched and read directly, Finding 1
- [`PortfolioAccountType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/PortfolioAccountType.java) — fetched and read directly, Finding 1
- [`StandingInstructionType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/StandingInstructionType.java) — fetched and read directly, Finding 1
- [`StandingInstructionStatus.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/StandingInstructionStatus.java) — fetched and read directly, Finding 4
- [`StandingInstructionPriority.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/StandingInstructionPriority.java) — fetched and read directly, Finding 6
- [`AccountTransferRecurrenceType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/account/domain/AccountTransferRecurrenceType.java) — fetched and read directly, Finding 2
- [`PeriodFrequencyType.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-core/src/main/java/org/apache/fineract/portfolio/common/domain/PeriodFrequencyType.java) — fetched and read directly, Finding 2
- [`StandingInstructionApiConstants.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-account-transfer/src/main/java/org/apache/fineract/portfolio/account/api/StandingInstructionApiConstants.java) — fetched and read directly, Finding 3
- [`AccountTransfersApiConstants.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-account-transfer/src/main/java/org/apache/fineract/portfolio/account/api/AccountTransfersApiConstants.java) — fetched and read directly, Finding 3
- [`AccountTransferStandingInstruction.java`](https://raw.githubusercontent.com/apache/fineract/develop/fineract-account-transfer/src/main/java/org/apache/fineract/portfolio/account/domain/AccountTransferStandingInstruction.java) — fetched directly (tool returned a structured summary of real file content), Findings 3, 4, 7
- [`get-page-items-standing-instruction-swagger.ts`](https://app.unpkg.com/fineract_sdk@0.0.5/files/models/get-page-items-standing-instruction-swagger.ts) and [`get-account-transfers-page-items.ts`](https://app.unpkg.com/fineract_sdk@0.0.5/files/models/get-account-transfers-page-items.ts) — fetched and read directly (OpenAPI-generated SDK model files), Finding 3
- Fineract database docs — fetched and read directly: [`m_account_transfer_standing_instructions`](https://fineract.apache.org/docs/database/tables/m_account_transfer_standing_instructions.html), [`m_account_transfer_standing_instructions_history`](https://fineract.apache.org/docs/database/tables/m_account_transfer_standing_instructions_history.html), [`m_account_transfer_details`](https://fineract.apache.org/docs/database/tables/m_account_transfer_details.html) — Findings 1, 2, 3, 5
- [apache/fineract PR #6826, "FINERACT-3121: Move account transfers into a fineract-account-transfer module"](https://github.com/apache/fineract/pull/6826) — fetched and read directly, Finding 8
- [apache/fineract PR #6806, "Fix rollback of all standing instructions when one fails"](https://github.com/apache/fineract/pull/6806) — fetched and read directly, Finding 5
- [apache/fineract PR #6540, "FINERACT-2876: Serialize standing instruction template options"](https://github.com/apache/fineract/pull/6540) — fetched and read directly, Finding 6
- [apache/fineract PR #6727, "FINERACT-2968: Use bind parameters for standing instruction history insert"](https://github.com/apache/fineract/pull/6727) — fetched and read directly, Finding 5
- [apache/fineract-consumer-facing issue #239, "Scheduled and recurring transfers on Fineract standing instructions"](https://github.com/apache/fineract-consumer-facing/issues/239) — fetched and read directly, Finding 8
- [FINERACT-2914, "Execute Standing Instruction job fails or skips months when the recurrence day is 29-31"](http://www.mail-archive.com/issues@fineract.apache.org/msg30000.html) — fetched and read directly, Finding 2
- [FINERACT-2923, "Execute missed scheduler job runs after restart or resume"](http://www.mail-archive.com/issues@fineract.apache.org/msg31164.html) — fetched and read directly, Finding 8
- [FINERACT-2400 dev-list proposal, "intent to work + proposed approach (partial standing-instruction transfers)"](http://www.mail-archive.com/dev@fineract.apache.org/msg12501.html) — fetched and read directly, Finding 5
- [Finecko — "Standing Instructions in Apache Fineract: Auto-Debit & Recurring Transfers"](https://finecko.com/docs/guides/standing-instructions.html) — fetched directly; used as a secondary, search-summarized source, and explicitly **corrected** in Findings 1 and 4 against the primary Java-enum sources (over-counted `instructionType` at 3 values against a real 2; claimed a persisted `status=4` "Expired" value not present in the real enum; a 1-indexed `recurrenceFrequency` scheme inconsistent with the directly-fetched `PeriodFrequencyType` enum's own 0-indexed values)
- apache/fineract PR #6235 ("Fix standing instruction not created at loan disbursement"), PR #6078/#6034 (Standing Instructions migration/partitioning), PR #5895 (dues-based optional fields) — titles/existence via search-engine summary, not independently fetched in full, Finding 2
- Internal: `research/fineract-core-functionality-gap-analysis.md` (T-059, Area 7) — built on directly for the original two-part-split finding, not re-derived
- Internal: `ARCHITECTURE.md` (read directly — Section 9's P-3 entry, the `ScheduleApi` service-map row)

## What was attempted and blocked or unreadable (stated plainly)

- `https://raw.githubusercontent.com/apache/fineract/develop/fineract-provider/src/main/java/org/apache/fineract/portfolio/account/api/StandingInstructionApiConstants.java` —
  **404 Not Found.** This was the file's pre-FINERACT-3121 location; the module move relocated it to
  `fineract-account-transfer/...`, which did fetch successfully and is cited above. Named here so the dead
  path isn't silently repeated by a future researcher.
- `https://github.com/apache/fineract/blob/develop/fineract-provider/src/main/java/org/apache/fineract/portfolio/account/api/StandingInstructionApiConstants.java` —
  **404 Not Found**, same stale pre-move path, GitHub `blob` view this time rather than `raw`.
- `https://raw.githubusercontent.com/apache/fineract/develop/fineract-account-transfer/src/main/java/org/apache/fineract/portfolio/account/domain/AccountTransferStandingInstructionsHistory.java` —
  **404 Not Found.** The real history-related Java class could not be located at this guessed path; this
  research instead relies on Fineract's own directly-fetched database-documentation page for the
  `m_account_transfer_standing_instructions_history` table (which did fetch successfully) plus a
  search-engine-converged description of the insert statement from a real, merged PR (#6727) for the
  equivalent field-level detail — graded accordingly (Finding 5's confidence notes this explicitly).
- `https://sourcegraph.com/github.com/apache/fineract/-/search?q=StandingInstructionApiConstants` —
  **404 Not Found.** Sourcegraph's own search UI did not resolve for this query via this toolchain; no
  fallback to this specific tool was pursued further since the raw GitHub fetch of the real file succeeded
  independently via a different, correct path.
- The exact literal `status` string values stored in `m_account_transfer_standing_instructions_history` —
  not independently confirmed from any directly-read source; every source converges on the underlying
  concept (success/partial/failure) but no single source gave an authoritative literal string list, named
  plainly as unresolved in Finding 5 and "What this does not settle" rather than guessed at.
- The exact current-`develop`-branch confirmation of priority-based execution ordering (Finding 6) — the one
  specific claim found (an `ORDER BY ... priority DESC` clause) traces partly through search results that
  also surfaced unofficial third-party forks (`COG-GTM/fineract-Core-Banking1`, a `devin-ai-integration[bot]`
  PR) rather than a confirmed fetch of `apache/fineract`'s own current source for this specific query —
  named plainly as unresolved rather than treated as settled.
- A `build.gradle`/dependency-manifest confirmation that Fineract's job framework is literally built on
  Spring Batch (Finding 8) — not independently fetched; the "Tasklet"/"partitioned job" vocabulary found in
  two directly-fetched PR descriptions is suggestive, not a confirmed dependency declaration.
