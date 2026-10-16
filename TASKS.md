# Board

Status: `todo` / `ready` / `doing` / `review` / `done` / `blocked`. Only the coordinator and owner edit this file; agents report in their PR and task file.

## A. Decisions for the owner (agents are blocked until these are answered)

| ID | Decision | Recommendation | Your answer |
|---|---|---|---|
| D-1 | **Demo slice for recruiters.** What must work end to end in Swagger on Azure? | Open account, fund, transfer with fee, reverse. Ledger (with chains) + minimal Orchestrator + minimal Identity. Product, Schedule, Financing, Channel come later. | |
| D-2 | Confirm P-1 to P-6 as Decided (or change them). P-13 is built (LedgerApi #43): mark Decided. | Confirm all | Confirmed, 16 Oct 2026, after T-011 research check against industry patterns (`research/architecture-patterns-p1-p6.md`) |
| D-3 | Repositories | Keep `LedgerApi`; add `garbati-platform` (this repo); one repo per new service | |
| D-4 | Contracts as OpenAPI files in `contracts/`; builders may not change them | Yes | |
| D-5 | Only the coordinator assigns decision numbers (next free: #48 after #45 to #47 are reserved) | Yes | |
| D-6 | Token (P-12): ship demo without it first, leave a marked seam (#47 later)? | Without it first | |
| D-7 | Section 9 open questions: leg purpose codes, max legs (10?), reversal when recipient spent funds, does a reversal restore the limit allowance | Principal/Fee/Tax; 10; whole reversal fails; yes | Ratified as built, 16 Oct 2026 |
| D-8 | Shared code between services (idempotency, audit, error shape) | Copy now, extract a package later | |
| D-9 | Identity design v2 changes (decisions table, token endpoint, scope split, Closed/Deceased cascade, freeze ownership, one active mandate index) | Apply the list from the review | Review located (`~/Downloads/Garbati_IdentityApi_Design.md`, 489 lines); T-013 checks it against industry standards; owner still needs to resolve 3 real gaps + ~11 "Proposed" (not yet Confirmed) sub-decisions inside it — see coordinator's note in chat, 16 Oct 2026 |
| D-10 | Hosting for new services (P-6) | Orchestrator + Product in one deployable at first | |
| D-11 | Human-only gates (AGENTS.md) are the right list | Yes | |
| D-12 | Quiz-before-merge rule: you answer QA's five questions before merging | Yes | |

## B. Tasks

| ID | Task | Owner agent | Depends on | Status |
|---|---|---|---|---|
| T-000 | Create `garbati-platform` repo from this folder; push to GitHub; add `templates/ci-dotnet.yml` to LedgerApi as `.github/workflows/ci.yml`; turn on branch protection and required checks | You | | done — repo created/pushed; LedgerApi CI predates this board (PR #18); branch protection on for both repos 16 Oct 2026 (LedgerApi requires `test`+`docker` checks; garbati-platform has no CI yet, so force-push/deletion protection only) |
| T-001 | LedgerApi: chains of legs (decision #45). See `tasks/T-001-ledger-chains.md` | builder | T-000, D-7 | done — built/merged as LedgerApi PR #26/#27 ahead of this board; QA-approved and owner-ratified 16 Oct 2026 |
| T-002 | QA review of T-001 PR | qa | T-001 | done — Approve, no blockers; `qa/reports/T-001.md` |
| T-003 | Update `ARCHITECTURE.md` decisions statuses after D-2 | coordinator | D-2 | done — P-1 to P-6 and P-13 marked Decided; ScheduleApi retry-on-failure flagged as a new open question for when ScheduleApi is designed |
| T-004 | LedgerApi: fee-income and tax-payable system accounts per currency (decision #46). See `tasks/T-004-ledger-fee-tax-accounts.md` | builder | T-001 | done — merged LedgerApi PR #28, 16 Oct 2026 |
| T-012 | QA review of T-004 / PR #28 | qa | T-004 | done — Approve, no blockers, 2 nits; `qa/reports/PR-28.md` |
| T-005 | LedgerApi: decision-token verification (decision #47) | builder | T-001, D-6 | blocked |
| T-006 | Identity design v2 (apply D-9) and new `Garbati.IdentityApi/docs/DESIGN.md` | coordinator + you | D-9 | todo |
| T-007 | Research: Sharia-compliant microfinance product rules (what is prohibited, how fees and late charges are treated). Output `research/sharia-product-rules.md`; needs scholar confirmation | researcher | | done — brief written; flags a sourcing disagreement on late-fee/ta'widh handling; **still needs a qualified scholar's sign-off before any rule is built on it** |
| T-008 | Research: Nigerian regulatory requirements relevant to a microfinance core system (CBN rules, tiered KYC, record keeping) | researcher | | done — brief written; 2 of several CBN documents read in full, others blocked (403)/image-only and rely on secondary sources; 7 items flagged **needing compliance-officer confirmation** |
| T-009 | Product questions-only design skeleton (first Sharia review point) | coordinator + you | T-007 | todo |
| T-010 | Orchestrator minimal design (Identity + Ledger only) | coordinator + you | T-006 | todo |
| T-011 | Research: industry-standard patterns behind P-1 to P-6 (Saga orchestration, database-per-service, modular-monolith-first), to inform D-2 | researcher | | done — all six match named patterns; `research/architecture-patterns-p1-p6.md` |
| T-013 | Research: check the IdentityApi design draft (`~/Downloads/Garbati_IdentityApi_Design.md`) against JWT/JOSE, OAuth2 local-verification and mandate-revocation standards, to inform D-9 | researcher | | done — token design matches RFC 8725/9068/7009 + OWASP closely; `research/identityapi-design-review.md` |

## C. Your own open items (not for agents)

- Confirm the live Azure URL for your README and LinkedIn.
- Run on Azure: `SELECT count(*) FROM transactions WHERE initiated_by IS NULL;`
- Set a removal date for the legacy-hash fallback in decision #44.
- Commonwealth Master's: deadline 20 Oct 2026; needs your degree class.
- Job application placeholders (years of experience, SQL Server, phone); verify the job poster before sending your CV.
