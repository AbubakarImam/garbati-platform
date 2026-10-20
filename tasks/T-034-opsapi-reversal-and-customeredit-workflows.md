# T-034: OpsApi — reversal requests and customer-detail-edit maker-checker workflows

Target repo: `Garbati.OpsApi` (foundation merged, T-033). Branch: `feat/reversal-and-customer-edit`. Spec: `docs/DESIGN.md` Sections 4.1 and 5 — read both in full first, plus Section 3's downstream-client table (just updated to grant `orchestrator.reversals`, D-19's context note added near decision #2).

**This is the second of two build tasks.** T-033 built everything except `docs/DESIGN.md` Section 6 groups 2 (customer-detail updates) and 3 (reversal requests) — the two stateful maker-checker workflows. This task builds both.

## The one real constraint this task works around

While scoping this task, `garbati-platform` D-19 found that `IdentityApi` has **no endpoint to apply a customer-detail edit** — only `POST /v1/customers` (create) exists; there's nothing to call on approval. A new `IdentityApi` endpoint (`garbati-platform` T-035, `PATCH /v1/customers/{customerId}/details`) has been approved but **may or may not be built yet** — check `garbati-platform/TASKS.md` for T-035's status before you start, but do not block on it:

- **The reversal-request workflow is fully executable today** — `OrchestratorApi POST /v1/reversals` already exists (T-030, live-verified). Build it completely, end to end, including the actual execution call.
- **The customer-detail-edit workflow stops at `Approved`** — build the entire staging/approval maker-checker workflow and test it completely, but do **not** call any `IdentityApi` endpoint to apply the change yet, even if T-035 has landed by the time you read this. Applying the change once an endpoint exists is a small, separate follow-up (mirrors exactly how `OpsReversalRequest` itself stopped at `Approved` in T-033, before T-030 existed) — keep this task's scope to staging and approving only.

## What to build

### 1. `OpsReversalRequest` (`docs/DESIGN.md` Section 5) — fully executable

Entity, per Section 5.1's table exactly:

| Field | Notes |
|---|---|
| `Id` | Public id, `OPS-REV-XXXXXXXXXX`. |
| `OriginalTransactionReference`, `Amount`, `Currency` | Caller-supplied at creation (5.1's own note: `LedgerApi` has no lookup-by-reference endpoint to derive these from). |
| `ReasonCode` | One of `IdentityApi`'s `ReversalReasonCode` values (`DUPL`/`TECH`/`FRAD`/`CUST`/`UNAUTH`) — validate against this exact list, reject anything else with 400. |
| `RequestedByStaffId` | The creating session's `staffId`. Must hold `Teller` or `CustomerServiceOfficer` (enforced by the create endpoint's role check). |
| `Status` | `PendingApproval` → `Approved`/`Rejected` → `Executed`/`ExecutionFailed`. `Expired` after 24 hours in `PendingApproval` (5.1). |
| `ApprovedByStaffId` | Nullable. Hard rule, enforced in code at the moment of decision: must not equal `RequestedByStaffId`, and the approving session's **current** role must be `Supervisor` — re-check the session's role at the moment the approve/reject action runs, not a value the browser might have cached from an earlier page load (5.1's "re-checked live, not just at queue-display time"). Any `Supervisor` qualifies (5.2) — not strictly the requester's own named `SupervisorStaffId`. |
| `RejectionReason` | Required if `Status = Rejected`. |
| `RequestedAt`, `DecidedAt`, plus an `ExecutedReference`/`ExecutionDetail`-shaped field (your naming) to hold `OrchestratorApi`'s own reference/decision id once executed, for traceability. |

Endpoints (`/v1/ops/reversals`):
- `POST` — create. `Teller`/`CustomerServiceOfficer` only. Requires a client-supplied `IdempotencyKey` (reuse `Validation/IdempotencyKeyRule.cs`, already built in T-033 — **do not** repeat T-033's original mistake of generating one server-side; this is a request that eventually moves money, it must be retry-safe from the moment it's created).
- `GET` (list) and `GET {id}` — role-scoped visibility, not a flat all-or-nothing gate: `Teller`/`CustomerServiceOfficer` see only requests **they themselves created**; `Supervisor` sees every `PendingApproval` request plus every request they've personally decided; `ComplianceOfficer`/`RiskOfficer`/`Auditor` get full read access to every request, any status (5.1's "read-only oversight," Section 4). `Administrator` has no access to this endpoint group at all (not in Section 4's row for this action). Enforce this as application-layer filtering inside the service, not just a single `[Authorize(Roles=...)]` gate — the roles allowed to call the endpoint at all is a wider set than what each of them is allowed to actually see.
- `POST {id}/approve` / `POST {id}/reject` — `Supervisor` only. `reject` requires `RejectionReason` in the body. On `approve`:
  1. Call `OrchestratorApi POST /v1/reversals` (new method on `IOrchestratorApiClient` — `orchestrator.reversals` scope, already granted in `docs/DESIGN.md` Section 3) with `OriginalTransactionReference`, `Amount`, `Currency`, `ReasonCode` from the stored row, and **`RequestedBy` set to the original `RequestedByStaffId`** — not the approver's own id. The approver is this workflow's own internal checker; `RequestedBy` is "who is actually asking for this reversal," which is the original requester, consistent with how `OrchestratorApi`'s own `ReversalRequest` field is used elsewhere on this platform.
  2. Use a **deterministic** idempotency key derived from this `OpsReversalRequest`'s own `Id` (e.g. `$"ops:{request.Id}"`), not a fresh one per call — so a browser retry of the approve action itself doesn't create two separate execution attempts at `OrchestratorApi`. This mirrors `OrchestratorApi`'s own pattern of deriving deterministic downstream keys from its own operation id.
  3. Store the result: `Executed` on success, `ExecutionFailed` on a business-rule failure (422) or any other error from the call — either way, record whatever reference/decision id/failure detail `OrchestratorApi` returned, and do not let a failure to record that detail mask the original call's own outcome (same principle `OrchestratorApi`'s own `ReversalService` already applies to its downstream call to `IdentityApi` — read `Garbati.OrchestratorApi/src/Garbati.OrchestratorApi/Services/Implementations/ReversalService.cs` for the shape, if it's checked out alongside this repo).
- **Lazy-expiry-aware listing**: a `PendingApproval` row already past its 24-hour window must not appear as actionable in the list endpoint, learning directly from `Garbati.IdentityApi`'s own T-031 QA finding (a stale-but-displayed-as-pending row is a real UX gap, even though approving it would still correctly fail) — exclude it from "pending" results via a plain read-only query condition, the same fix shape T-031 used, not a write.

### 2. `OpsCustomerEditRequest` (`docs/DESIGN.md` Section 4.1) — stops at `Approved`

Entity, same shape discipline as `OpsReversalRequest` above:

| Field | Notes |
|---|---|
| `Id` | Public id, `OPS-EDIT-XXXXXXXXXX`. |
| `TargetCustomerId` | The customer this edit would apply to. |
| `ProposedFullName`, `ProposedEmail`, `ProposedPhoneNumber` | All nullable — a partial update, matching `IdentityApi`'s own new `UpdateCustomerDetailsRequest` shape (T-035) field-for-field, not a generic untyped JSON blob. At least one must be non-null at creation (400 if all three are null — nothing to propose). |
| `RequestedByStaffId` | Must hold `CustomerServiceOfficer`. |
| `Status` | `PendingApproval` → `Approved`/`Rejected` → `Applied`/`ApplicationFailed`. Define the full enum now (matching the design's "same shape" framing), but **no code path in this task reaches `Applied`/`ApplicationFailed`** — `Approved` is this task's terminal state for this entity. Say this explicitly in the PR so it isn't mistaken for an oversight. |
| `ApprovedByStaffId` | Same hard rule as reversals: must not equal `RequestedByStaffId`, must currently hold `Supervisor` (any `Supervisor`, 4.1), re-checked live at decision time. |
| `RejectionReason` | Required if `Status = Rejected`. |
| `RequestedAt`, `DecidedAt` | |

Endpoints (`/v1/ops/customer-edits`):
- `POST` — create. `CustomerServiceOfficer` only.
- `GET` (list) and `GET {id}` — per Section 4's table for this action, **only** `CustomerServiceOfficer` (their own submitted requests only) and `Supervisor` (the full queue) have any access at all — unlike reversals, there is no read-only-oversight row for `ComplianceOfficer`/`RiskOfficer`/`Auditor` on this one. Every other role, including `Administrator`, gets `403`.
- `POST {id}/approve` / `POST {id}/reject` — `Supervisor` only, same hard rule as reversals. On `approve`, set `Status = Approved` and stop — **do not call `IdentityApi`**, even though `identity.customer.write` is already granted to this service's `IdentityApi` client (it's granted for this future step, deliberately unused by this task — the existing `IIdentityApiClient` interface's own comment already says so). `reject` requires `RejectionReason`.
- Same lazy-expiry-aware listing treatment as reversals — pick a sensible expiry window for `PendingApproval` (24 hours, matching reversals, unless you have a specific reason to choose differently — say so in the PR if you do) and exclude expired-but-untouched rows from the list the same way.

## Scope

- Two new entities + one migration, `Services/Implementations/{ReversalRequestService,CustomerEditRequestService}.cs` + interfaces, two new controllers, request/response contracts for both, the new `IOrchestratorApiClient.ReverseAsync` method and its `Clients/Contracts/Orchestrator/{ReversalRequest,ReversalResultResponse}.cs` mirrors of `OrchestratorApi`'s real contract (read its actual `Contracts/Requests/ReversalRequest.cs`/`Contracts/Responses/ReversalResultResponse.cs` directly, don't guess the shape).
- New `OpsRoles` constant groupings as needed for the access patterns above (the existing file already has `TellerOrSupervisor`/`SupervisorOrAuditor`/`AnyRole` as a precedent for how these are named).
- Tests: creation (happy path, missing idempotency key/reason-code/proposed-fields validation), role-scoped list visibility (prove a `Teller` cannot see another `Teller`'s reversal request; prove a `ComplianceOfficer` can see everything; prove `Administrator`/every-other-role gets `403` on customer-edits), the hard maker-checker rule (requester cannot approve their own request; a non-`Supervisor` approver is rejected; re-checked live, not from a stale cached value), the deterministic-idempotency-key behavior on the `OrchestratorApi` reversal-execution call (a regression test capturing the actual key sent, the same class of test T-024/T-030/T-033 all needed for this exact bug shape), the lazy-expiry list exclusion (mirroring T-031's own regression test for this), and that `OpsCustomerEditRequest` genuinely never calls `IdentityApi` on approval (assert the fake `IIdentityApiClient`'s relevant call count stays zero throughout).

## Out of scope

- Applying a `OpsCustomerEditRequest` to `IdentityApi` — a small, separate follow-up once T-035 is confirmed built and deployed; not this task.
- Anything about the ops-console frontend (`garbati-ops-console`, T-029 design-only) — this task only builds the API surface it will eventually call.
- Re-litigating any decision already ratified in `docs/DESIGN.md` Sections 4.1/5 (e.g. "any Supervisor, not the named one" — settled, not open here).

## PR

Open a pull request from `feat/reversal-and-customer-edit` against `main` on `AbubakarImam/Garbati.OpsApi` (branch-protected — get `test`/`docker` CI green on your own branch first). Include what changed, how it was tested, every judgment call this task file left open (in particular, double-check whether T-035 has merged by the time you open this PR, and say so explicitly either way), and a draft LEARNING_LOG-style plain-language explanation plus five interview questions with answers. Do not merge — only the owner merges.
