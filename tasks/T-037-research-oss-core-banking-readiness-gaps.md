# T-037: Research — what does GARBATI need to be a credible, standard open-source core-banking platform?

## Context

GARBATI has grown into five real services (`LedgerApi`, `Garbati.IdentityApi`, `Garbati.OrchestratorApi`, `Garbati.StaffIdentityApi`, `Garbati.OpsApi`) plus a frontend design, all built with real money-safety discipline — idempotency, audit trails, maker-checker, QA review on every merge. But nothing in this project has yet asked the separate question: **what does an open-source project in the core-banking space need to have, beyond working code, before a stranger would trust it enough to run it, contribute to it, or build a real institution on it?**

Already confirmed missing (checked directly, not assumed) at the `garbati-platform` repo root: no `LICENSE` file, no `CONTRIBUTING.md`, no `CODE_OF_CONDUCT.md`, no `SECURITY.md`, no `CHANGELOG.md`. This is the starting evidence, not the full picture — your job is to find everything else, not just confirm this list.

## What to research

**1. Benchmark against real, comparable open-source core-banking/fintech projects.** Name real, citable projects — Apache Fineract (formerly Mifos X) is the closest direct comparison (open-source core banking, used by real microfinance institutions) and should be looked at in real detail; also look at others if relevant (Cyclos, OpenCBS/Kobocollect-adjacent tools, or general-purpose fintech-infrastructure projects like Formance/Midaz if they're a closer match for a multi-service ledger architecture). For each, identify:
   - What governance/community files they ship (`LICENSE`, `CONTRIBUTING`, `CODE_OF_CONDUCT`, `SECURITY.md`/responsible-disclosure policy, `CHANGELOG`, issue/PR templates, a roadmap).
   - What license they actually chose, and why that choice matters for a banking-adjacent project specifically (e.g., Apache 2.0's patent grant vs. a copyleft license's effect on an institution wanting to run a modified, closed fork — this is a real decision point GARBATI's owner will eventually have to make, flag it as such, don't make it for them).
   - How they document security practices and vulnerability disclosure (does a security researcher have a clear, safe channel to report a bug before it's public? Core-banking software is an unusually high-value target — check whether comparable projects treat this differently from an ordinary OSS project).
   - What compliance/regulatory posture they document (do they claim PCI-DSS-adjacent practices, SOC2, ISO 27001, or explicitly disclaim them as "the operator's responsibility"?).

**2. What does a real banking-adjacent open-source project need operationally that GARBATI doesn't have yet?** Check specifically for:
   - Secrets management strategy — GARBATI currently manages Azure App Service secrets by hand (`az webapp config appsettings set`); is there a standard open-source pattern (Vault, sealed secrets, environment-based convention) worth documenting as a recommendation, even if not built yet?
   - Infrastructure-as-code — GARBATI's deployment is currently a hand-run sequence of `az` CLI commands recorded in GitHub Actions; do comparable projects ship Terraform/Bicep/Pulumi, or is a CI-script-based approach actually normal at this stage for a project this size?
   - A documented backup/disaster-recovery story for the ledger database specifically — this is the one dataset in the whole platform that must never be silently lost; what do comparable systems document (or fail to document) here?
   - Observability beyond request logging — structured metrics, tracing, or a reference Grafana/Prometheus setup. Is this something real open-source core-banking projects ship, or something every adopter is expected to bring themselves?
   - A local, single-command "run the whole stack" developer experience (e.g., one `docker-compose.yml` spinning up all five services plus their databases together) — GARBATI currently has per-service Docker setups but nothing that runs the whole platform together. Check whether this is standard practice for a multi-service OSS project.

**3. What's missing specifically for the *banking* domain, not just "OSS project hygiene"?** Check:
   - Standard banking messaging/interchange formats (ISO 20022, or any Nigerian-specific interbank standard relevant to NIBSS/CBN) — does GARBATI need to speak any of these to be taken seriously, or is that correctly out of scope until a real bank integration is attempted?
   - Regulatory reporting — do comparable core-banking systems ship any built-in reporting/statement-generation capability, or is this always left to the adopter? (`garbati-platform/research/nigerian-regulatory-requirements.md` already exists — read it first so you don't repeat work already done; this question is about *tooling* to meet those requirements, not re-researching the requirements themselves.)
   - Data residency/encryption-at-rest for regulated customer data — `Garbati.IdentityApi`'s own `docs/DESIGN.md` already flags that BVN/NIN storage/encryption was deliberately left undesigned (T-016's own flagged follow-up). Confirm whether this is still unaddressed, and whether comparable open-source projects in this space document a specific, concrete pattern for this (not just "encrypt sensitive data" as a platitude).

**4. What's missing for *contributor* readiness, separate from operator readiness?** A project can be production-ready for the owner's own use and still be unusable for an outside contributor to safely participate in. Check:
   - Is there a clear, written explanation of the multi-repo structure (`garbati-platform` + 5 service repos) anywhere a newcomer would actually find it? (The existing `README.md`/`ARCHITECTURE.md` are written for the owner and the coordinator process, not necessarily for an outside contributor meeting the project for the first time — read them and assess honestly whether they'd orient a stranger.)
   - Is there a documented, minimal path to "clone this, run it locally, make one small change, open a PR" for someone who is not already deeply familiar with GARBATI's coordinator/agent-driven workflow (`AGENTS.md` is written for AI builder agents specifically — would a human contributor need something different)?

## Output

Write the brief to `research/oss-core-banking-readiness-gaps.md`, matching this repo's existing research-brief style (cited sources, clear findings, an explicit "what I could not verify" section). Structure it as a **prioritized gap list**, not just a survey — for each gap, say concretely what's missing, why it matters for *this specific kind* of project (open-source + core-banking, not open-source in general), and a rough sense of effort (a single file to write vs. a real engineering task). Flag clearly which gaps are the owner's decision to make (e.g., license choice) versus which are just omissions that can be filled in directly once approved (e.g., writing a `CONTRIBUTING.md`).

**Do not write any code or any of the actual governance files yet** — this is a research/gap-analysis brief only. The coordinator will bring the findings back to the owner, the same way every other research brief on this platform has fed a subsequent decision before anything gets built or written.

## Out of scope

- Writing `LICENSE`/`CONTRIBUTING.md`/`SECURITY.md`/etc. yourself — that's a follow-up task once the owner has seen this brief and made the license/governance-model decisions it will surface.
- Re-researching Nigerian regulatory requirements or Sharia product rules — those briefs already exist (`research/nigerian-regulatory-requirements.md`, `research/sharia-product-rules.md`); read them for context, don't redo them.
- Any code change to any service.
