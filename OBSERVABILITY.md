# Observability

T-040 (G10, from `research/oss-core-banking-readiness-gaps.md`): what GARBATI gives you today for
understanding what's happening across the five services, and what it deliberately does not — stated
explicitly, rather than left for an operator to discover by looking for a dashboard that doesn't exist.

## What exists today, confirmed per service

Every one of the five services (`LedgerApi`, `Garbati.IdentityApi`, `Garbati.OrchestratorApi`,
`Garbati.StaffIdentityApi`, `Garbati.OpsApi`) follows the same three conventions
(`ARCHITECTURE.md` Section 5), confirmed directly against each one's real source, not assumed to be
uniform just because the pattern started in one service and got copied:

- **Structured request logs (Serilog).** One line per request — method, path, status code, duration,
  correlation id, calling client, and (for money-moving requests) the idempotency key. Request bodies are
  never logged; `ARCHITECTURE.md` Section 5 also forbids logging identity numbers (BVN/NIN), documents or
  keys. Configured per service in its own `appsettings.json` `Serilog` section; written to the console,
  which Azure App Service captures as its own log stream.
- **A correlation id on every call.** `X-Correlation-ID`, generated if the caller doesn't supply one,
  echoed back on the response, included in every error body, and propagated to every downstream call a
  service makes (confirmed: every service that calls a sibling has its own
  `CorrelationIdPropagationHandler`/`CorrelationIdMiddleware`). This is what actually lets you trace one
  request across service boundaries today — by grepping each service's own logs for the same id — not a
  single-pane trace view.
- **A durable, transactional audit trail — with one real, deliberate exception.** `LedgerApi`
  (`audit_logs`), `Garbati.IdentityApi`, `Garbati.OrchestratorApi` and `Garbati.StaffIdentityApi`
  (`StaffAuditLog`) each write an audit row in the same database transaction as the change it describes.
  `Garbati.OpsApi` is the one service that does **not** have its own durable audit table for its own
  session lifecycle — this was a deliberate choice made when it was built (`docs/DESIGN.md` decision
  under Section 2: "relies on the existing correlation-id-tagged request logging ... if session lifecycle
  needs a durable, queryable audit trail beyond request logs ... that is an open question for the owner,
  not decided in this task"), not an oversight being surfaced for the first time here. It remains open.

This is already more than a bare-minimum logging story, and — per the benchmark research in
`research/oss-core-banking-readiness-gaps.md` — roughly in line with what a comparable real OSS
core-banking project documents (Apache Fineract's own docs leave metrics/tracing to the operator too).

## What does not exist today

- **No metrics.** No request-rate, latency-percentile, or error-rate dashboard for any service. If a
  service is slow or erroring more than usual right now, nothing surfaces that on its own — you'd find out
  from a user report, or from reading request logs after the fact.
- **No distributed tracing.** The correlation id lets you manually follow one request's logs across
  services; there is no tool that assembles that automatically into a single visual trace, no span
  timing breakdown of where time was spent across a multi-service call.
- **No alerting.** Nothing pages anyone when something goes wrong. `/health/live` and `/health/ready`
  exist on every service (for a platform or load balancer to check), but nothing is currently watching
  them and notifying a human.

## The stated posture: bring your own

**GARBATI does not ship a metrics/tracing/alerting stack, and does not plan to until there's a concrete
reason to.** This is a deliberate posture, not silence — the same one Apache Fineract itself takes (its
own documentation leaves monitoring infrastructure to the deploying operator). Stated explicitly so it's
a named choice an operator can act on, rather than an absence they have to go discover:

- If you're running GARBATI and want metrics/tracing/alerting, the realistic options given today's stack
  (Azure App Service, ASP.NET Core, Serilog) are **Azure Application Insights** (lowest integration
  effort — an App Service add-on, largely config rather than code, and already in the same cloud every
  service is deployed to) or a self-hosted **Prometheus + Grafana** pair (more operational work, not tied
  to Azure). Neither is set up today; this document does not pick one, since that choice depends on who's
  actually operating this and what they're already paying for elsewhere — an operator decision, not a
  default GARBATI makes for you.
- The correlation id and the audit trail are deliberately kept independent of whichever
  metrics/tracing stack you choose — grepping logs by correlation id and querying the audit tables
  directly will keep working exactly as documented in each service's own README, with or without
  Application Insights or Prometheus layered on top.

## When this should actually get built

Treated as a later-tranche engineering task, not urgent at GARBATI's current size and traffic — revisit
when any of these becomes true:

- A real deployment has enough traffic that "read the logs after a user complains" stops being fast
  enough to catch problems.
- A specific incident actually needed a trace/metrics view that didn't exist, and the gap was felt for
  real, not hypothetically.
- An operator (possibly not the original owner) asks for it directly.

Until then, this document is itself the deliverable for G10 — stated expectation, not a built dashboard.
