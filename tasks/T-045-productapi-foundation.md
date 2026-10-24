# T-045: ProductApi foundation — build the whole v1 (minimal) design

Target repo: `Garbati.ProductApi` (new, empty except `docs/DESIGN.md`/`README.md`/governance files — already
created and pushed to `main` directly, no PR for that initial commit). Branch: `feature/productapi-foundation`.

## Context

Read `docs/DESIGN.md` in full first — it is the actual spec for this task, written and ratified by the
owner. This task builds everything in it: project skeleton, five entities, five endpoints, migrations, and
tests. Nothing in this task requires any Sharia or compliance decision that hasn't already been made —
`DESIGN.md` Section 0 explains exactly why `FeeRule`/`ProfitSharingTerms` are built as empty shapes, not
decisions for you to second-guess or fill in.

**Project scaffold**: mirror `Garbati.StaffIdentityApi`'s scaffold exactly (the simplest sibling service —
calls nothing else, same as this one) — `global.json`, `dotnet-tools.json` (the `dotnet-ef` tool manifest),
`.gitignore`, `.dockerignore` (same pattern, adjusted for this project's name — remember the
`appsettings.Development.json` exclusion line, confirmed elsewhere this session to be deliberate, not an
oversight), the two-project `.sln` (src + tests), the same `Dockerfile` four-stage shape (`build` → `bundle`
→ `migrator` + `final`), and the same `.github/workflows/ci.yml` (`test` + `docker` jobs, no `deploy` job
yet — this service isn't being wired to Azure in this task). Copy `Microsoft.Extensions.Diagnostics.
HealthChecks.EntityFrameworkCore`, `Npgsql.EntityFrameworkCore.PostgreSQL`, `EFCore.NamingConventions`,
`Serilog.AspNetCore`, `Swashbuckle.AspNetCore`, `Microsoft.EntityFrameworkCore.Design` package references
at the same pinned versions `Garbati.StaffIdentityApi` uses. No new package beyond what every sibling
service already depends on — this task introduces no new dependency.

## What to build

### 1. Entities (DESIGN.md Section 2)

- `Product` — `ProductCode` (string, e.g. `NG-SAV-STD`), `ProductName`, `JurisdictionCode`, `CurrencyCode`,
  `ProductType` (`Savings`/`Current` — a plain string column is fine, no enum table needed for two values),
  `EffectiveFrom`, `EffectiveTo` (nullable).
- `ProductLimitPolicy` — `ProductCode`, `MinimumBalance` (decimal), `AllowedOperations` (`List<string>`,
  stored as `jsonb` via the same `stringListConverter`/`HasColumnType("jsonb")` pattern
  `Garbati.IdentityApi`'s `KycTierPolicy.RequiredEvidence` already uses — read that entity and its
  `IdentityDbContext` configuration directly before implementing this, copy the pattern, don't reinvent
  it), `EffectiveFrom`, `EffectiveTo`.
- `AccountProductBinding` — `AccountNumber` (string — the `LedgerApi` account number; this service never
  calls `LedgerApi` to validate it exists, see Section 2.3), `ProductCode`, `BoundAt`.
- `FeeRule` — `ProductCode`, `OperationType`, `FeeType` (`Flat`/`Percentage`/`None`), `Amount` (decimal,
  nullable — meaningless when `FeeType` is `None`), `EffectiveFrom`, `EffectiveTo`. **No row is ever
  seeded for this table.**
- `ProfitSharingTerms` — `ProductCode`, `ProfitShareRatioCustomer` (decimal), `ProfitShareRatioInstitution`
  (decimal), `EffectiveFrom`, `EffectiveTo`. **No row is ever seeded. No endpoint in this task reads or
  writes this table at all** — it exists purely as a named table for a future task to extend, per
  `DESIGN.md` 2.5. Do not build any endpoint, service method, or validation touching this entity.

### 2. Migrations

- One migration creating all five tables.
- A second, separate seed migration for `ProductLimitPolicy` only — 2-3 placeholder rows (e.g. a `NG`
  `Savings` and a `NG` `Current` product's limits), with a comment at the top of the migration file
  copying the exact spirit of `Garbati.IdentityApi`'s `SeedPlaceholderKycTierPolicies` migration comment:
  state plainly these are placeholder values, not real confirmed Nigerian minimum-balance figures, and
  must be replaced before relying on this table for anything real. **Do not seed `FeeRule` or
  `ProfitSharingTerms` with anything, placeholder or otherwise** — leave both tables genuinely empty after
  migrations run. This is the one place in this task where "seed a placeholder like the sibling service
  did" does **not** apply — the whole point of Section 0's design decision is that these two tables start
  and stay empty until a real task (not this one) decides to fill them.

### 3. Endpoints (DESIGN.md Section 3)

All under `/v1/`, API-key + scope authenticated (same `ApiKeyAuthenticationHandler`/scope-policy pattern
every sibling service uses — read `Garbati.StaffIdentityApi`'s `Authorization/` folder for the simplest
reference implementation to copy). New scopes: `product.catalog.read`, `product.catalog.write`,
`product.pricing.read`.

- `POST /v1/products` (`product.catalog.write`) — create a `Product`. Validate required fields; `409` if
  `ProductCode` already has a currently-effective row (same "don't silently duplicate" discipline as
  every other create-something-new endpoint on this platform, though note this one has no idempotency-key
  requirement unless you judge the create-twice risk real enough to need one — state your reasoning in the
  PR either way, matching how `KycDecisionRequest`'s own doc comment explains why it skipped one).
- `GET /v1/products/{productCode}` — fetch the product plus its currently-effective `ProductLimitPolicy`
  (`MinimumBalance`, `AllowedOperations`) in one response. `404` if no product with that code exists at all;
  if the product exists but has no currently-effective limit policy, return the product with `null`/empty
  limit fields rather than a 500 — a product predating any limit policy being configured is a real possible
  state, not a bug.
- `POST /v1/account-bindings` (`product.catalog.write`) — create an `AccountProductBinding`. Requires a
  client-scoped idempotency key (this one **does** need one — it's the literal "create a new thing" shape
  every other idempotent-create endpoint on this platform already uses, unlike `POST /v1/products` above
  which is a rarer, ops-driven catalog action). `404` if `ProductCode` doesn't exist. Do **not** call
  `LedgerApi` to check the account number is real — per `DESIGN.md` 2.3, that's `OrchestratorApi`'s job,
  already done before this endpoint is ever reached; this service trusts the caller on that one fact.
- `GET /v1/account-bindings/{accountNumber}` — which product (if any) an account is bound to. `404` if
  no binding exists for that account number.
- `POST /v1/fee-quotes` (`product.pricing.read`) — request shape: `ProductCode`, `OperationType`, `Amount`.
  Look up a currently-effective `FeeRule` for that `(ProductCode, OperationType)`. **None will ever be
  found in this task's own tests or in any real deployment yet** — assert the response is a clear,
  explicit `{ "feeResult": "NoFeeConfigured" }` shape (your own naming, document it in the PR), never a 404,
  never a `0`-amount success, never an exception. If you want to also implement the "found a real `FeeRule`
  row and compute the fee" branch for completeness (since the table/lookup exists), that's fine — just
  make sure the untested, can't-currently-happen-for-real branch is clearly marked as such and the
  `NoFeeConfigured` branch is the one with real test coverage, since it's the only branch any real caller
  will hit until a future task seeds real fee rules.

### 4. Health checks, logging, correlation id

Same as every sibling: `/health/live`, `/health/ready` (checks the database), Serilog request logging,
`CorrelationIdMiddleware`, `X-Correlation-ID` on every response and error body. Copy
`Garbati.StaffIdentityApi`'s `Middleware/` folder directly — there is nothing service-specific about any
of this.

### 5. Audit trail

One audit row per `Product` creation and per `AccountProductBinding` creation, same "written in the same
database transaction as the change" convention every sibling service follows. No audit events needed for
`GET` endpoints or for the (always-`NoFeeConfigured`-today) fee-quote lookup, which changes nothing.

## Tests to write

- Migration applies cleanly; `ProductLimitPolicy`'s two placeholder rows exist afterward with the exact
  values the seed migration specifies; `FeeRule` and `ProfitSharingTerms` are provably empty tables after
  migration (a real `SELECT count(*)` assertion, not an assumption).
- `POST /v1/products` then `GET /v1/products/{productCode}` round-trips correctly, including its limit
  policy fields when one exists and when one doesn't.
- `POST /v1/account-bindings`: idempotency key behavior (same key + same body returns the same binding;
  same key + different body returns 409 — the standard pattern, test it the standard way); `404` for an
  unknown `ProductCode`.
- `POST /v1/fee-quotes` against a real, seeded product returns `NoFeeConfigured` — a direct assertion on
  the response shape, not just a 200 status code.
- Scope enforcement: each endpoint rejects a caller with the wrong scope (403) and no API key at all (401)
  — mirror `Garbati.StaffIdentityApi`'s own `AuthorizationTests.Endpoints`-style table-driven test for this.
- No test, anywhere, writes a row to `FeeRule` or `ProfitSharingTerms`, and no application code path does
  either outside the (if built) untested "real fee rule found" branch noted above.

## Explicitly out of scope

- Any real fee amount, tax rule, or profit-sharing ratio — covered at length above; this is the single
  most important boundary of this task.
- Wiring `OrchestratorApi` to actually call this service (account-opening step 5, a transfer's fee leg) —
  a separate, future task on `OrchestratorApi`'s side, not started here.
- Deploying this service to Azure — no `deploy` CI job in this task, matching how every sibling service
  started (a placeholder/absent deploy job until a later, separate task wires it up).
- Any endpoint or logic touching `ProfitSharingTerms` — the table exists, nothing else does.

## PR

Open a pull request from `feature/productapi-foundation` against `main` on `AbubakarImam/Garbati.ProductApi`
(branch-protected, no CI-required-checks configured yet since this is the first PR — get `test`/`docker`
green anyway before opening it). Describe what was built, and be explicit in the description about the two
empty tables and why, so a reviewer doesn't have to go hunting for whether that was deliberate. Do not
merge — only the owner merges.
