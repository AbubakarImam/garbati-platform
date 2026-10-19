# T-026 — Research: how do real core-banking platforms separate staff/ops identity from customer identity?

## Context

GARBATI is getting its first frontend: a React + TypeScript ops console — the bank's internal back-office
tool, used by bank staff (not customers) to look up customers/accounts, work the approvals queue (the
joint-account and reversal-authority workflows already built but never given a usable front end), run
manual account-opening/funding/transfer operations for testing, update account details, and manage staff
accounts and roles.

This console will sit behind a new backend-for-frontend service (owner's decision, already made — not
open for this research) that holds the real `LedgerApi`/`IdentityApi`/`OrchestratorApi` service API keys
server-side and gives staff a real login session instead of ever putting service credentials in the
browser.

**What's still genuinely open, and the one thing this research is for:** where should *staff* identity —
who is logging in, and what role/permissions they hold — actually live? Two candidates the owner is
choosing between:

1. A new, clearly-bounded area inside the existing `Garbati.IdentityApi` service (new `StaffUser`/`Role`
   entities, separate tables, same service/database) — no new service to build/deploy/pay for.
2. A brand-new, dedicated service, separate from `IdentityApi`.

`IdentityApi`'s entire existing design (`Garbati.IdentityApi/docs/DESIGN.md`) is built around *customers*:
KYC tiers, mandates, joint-account approvals — a fundamentally different kind of "identity" than "a bank
employee with a job-function role." Mixing the two into one data model without checking real precedent
first is exactly the kind of judgment call the owner shouldn't be asked to make blind.

## What to research

1. **How do real banks/core-banking platforms (Temenos, Mambu, Finacle, FIS, or similar — name real,
   citable systems/vendors where you can) separate "customer identity" from "staff/internal-user
   identity"? Is there a standard pattern (separate system entirely, separate bounded context in the same
   platform, directory-integrated like LDAP/Active Directory/SSO, something else)?**
2. **What roles does a typical bank back-office/ops console actually have?** Look for real, named
   examples — e.g. Teller/Cashier, Customer Service Officer, Compliance/KYC Officer, Risk/Operations
   Officer, Supervisor/Approver, Administrator, Auditor (read-only) — and what each role is typically
   allowed vs. forbidden to do. Flag any "separation of duties" conventions (e.g. the person who initiates
   a reversal cannot be the person who approves it) — GARBATI's reversal-authority design (IdentityApi
   `docs/DESIGN.md` Section 8) already assumes an ops channel exists with its own maker-checker, so check
   whether that assumption matches real practice.
3. **How does staff authentication typically work** — session cookies, short-lived JWTs, something else —
   for an internal back-office web app specifically (not a customer-facing app, not a public API)? Keep
   this concrete enough that a design task can act on it directly.
4. **What does a real core-banking platform's role model need to support that a simple flat
   "role = fixed permission set" model might miss?** (e.g. role scoped to a branch/jurisdiction, temporary
   elevated access, role changes requiring approval themselves.) Flag what's realistic for GARBATI's scale
   right now vs. what's a "note for later, don't build yet."
5. **Recommend, with reasoning:** should GARBATI's staff identity live inside `IdentityApi` as a separate
   bounded context, or as its own dedicated service? Give the owner a clear recommendation plus the honest
   trade-offs (cost/complexity vs. architectural cleanliness) — the same way `research/joint-account-approval-patterns.md`
   and `research/architecture-patterns-p1-p6.md` did for earlier decisions. This is input to the owner's
   decision, not the decision itself — don't present it as already settled.

## Output

Write the brief to `research/staff-ops-identity-patterns.md`, matching this repo's existing research-brief
style (see `research/joint-account-approval-patterns.md` or `research/architecture-patterns-p1-p6.md` for
the expected shape: cited sources, clear recommendation, explicitly flagged open questions/risks).

## Out of scope

- Do not design the actual `StaffUser`/`Role` schema — that's a follow-up design task once the owner
  picks a location.
- Do not touch the already-decided items: React + TypeScript for the frontend, a new backend-for-frontend
  service holding real service credentials, and the v1 feature scope (customer/account lookup, approvals
  queue, transfer/transaction testing console, account detail updates, staff & role management).
- Do not write any code.
