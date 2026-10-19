# T-031 — IdentityApi: add `GET /v1/authorization/requests` (list pending)

## Context

`garbati-platform` D-15, owner-approved 19 Oct 2026. Found while designing the ops console's backend-for-frontend (`Garbati.OpsApi`, T-028): `AuthorizationController` only has fetch-by-id (`GET /v1/authorization/requests/{id}`, `AuthorizationController.cs` line 118) — there is no way to list multiple pending requests, so an ops-console "approvals queue" screen has nothing to browse. This is a small, purely additive, read-only endpoint — it does not change any existing endpoint, scope, or behavior.

## What to build

**New endpoint**, `GET /v1/authorization/requests?status=Pending&pageSize=&pageToken=`:

- Scope: `identity.authorization.check` (the existing scope — `[Authorize(Policy = IdentityScopes.AuthorizationCheck)]`, same policy the existing `GetAuthorizationRequest`/`Approve` actions already use on this controller). **No new scope.**
- Query target: `PendingApprovalRequest` (the entity backing `GetPendingRequestAsync`, `JointApprovalService.cs` — `Entities/PendingApprovalRequest.cs`). A single-holder (`RequiredApprovals <= 1`) authorization request never creates a `PendingApprovalRequest` row at all (it resolves synchronously via `AuthorizationDecision` — see `AuthorizationController`'s own comment on `RequestAuthorization`), so this endpoint's job is specifically "pending **joint-approval** requests," not every authorization decision ever made. Name it accordingly in the response/docs so a caller doesn't expect single-holder transfers to show up here.
- `status` query param: defaults to `Pending` (`PendingApprovalRequestStatus.Pending`) if omitted — the actually-useful case for a queue. Accept any of the four enum values (`Pending`/`Approved`/`Expired`/`Cancelled`) for completeness, but don't build anything beyond a `WHERE Status = @status` filter — no new business logic.
- Paging: `pageSize` (default and max — pick sensible values, e.g. default 20/max 100, matching whatever paging convention this platform already uses elsewhere if one exists; if none exists yet, a simple `pageSize`/`pageToken` (opaque cursor, e.g. base64 of the last `Id`) is enough — don't build keyset/offset paging machinery beyond what a queue of a realistically small number of pending requests needs).
- Response: a list of the **existing** `PendingApprovalRequestResponse` shape (`Contracts/Responses/PendingApprovalRequestResponse.cs`) — reuse it exactly as-is, do not create a second, slightly-different shape for the list vs. the single-item lookup. Wrap in whatever this platform's established "paged list" envelope is (check `LedgerApi`/`OrchestratorApi` for an existing paged-list response convention before inventing a new one — if none exists yet, a plain `{ items: [...], nextPageToken: string? }` is fine and should be named consistently for whichever other list endpoint needs the same shape next).

## Scope

- One new controller action on `AuthorizationController` (or a new method on `IJointApprovalService` it calls — your call, consistent with how `GetPendingRequestAsync` is already structured).
- Tests: empty result, multiple pending requests returned correctly ordered (oldest-first or newest-first — pick one, document which, and be consistent), status filter actually filters, paging actually pages, and a request that resolved via the single-holder/no-`PendingApprovalRequest`-row path correctly does **not** appear (confirms the "joint-approval only" scope note above, not just documents it).
- Do not touch `GetAuthorizationRequest`, `Approve`, `RequestAuthorization`, or anything else on this controller — purely additive.

## Out of scope

- Any change to `AuthorizationDecision`/single-holder transfer visibility — this endpoint is about `PendingApprovalRequest` rows only, as explained above.
- Reversal requests — those live entirely in `Garbati.OpsApi`'s own `OpsReversalRequest` table (T-028's design), never in `IdentityApi` at all; nothing here lists them.
- Any new scope, any change to who may call this (same scope as the existing single-lookup action, by design).
