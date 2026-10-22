# T-041: Research — real Nigeria-based hosting options for LedgerApi's database (G4 follow-up)

## Context

`research/oss-core-banking-readiness-gaps.md` (T-037, G4) surfaced a new finding: **CBN Circular
PSS/DIR/PUB/CIR/001/004** (per secondary legal/industry sources — Legal500, TechCabal — primary circular
text not yet confirmed) reportedly requires Nigeria-originated payment/financial transaction data to be
stored and processed **inside Nigeria** by **1 January 2027**. That brief said this deadline was "over a
year out" — **this was a dating error, caught and corrected by the coordinator on re-check: 1 January
2027 is approximately 71 days (≈2.3 months) from today, 22 October 2026.** Treat this as more urgent than
the original brief framed it.

`LedgerApi` currently runs on **Azure App Service + Azure Database for PostgreSQL Flexible Server, both
in "South Africa North"** — confirmed directly against the live resource, not Nigeria. The coordinator
also confirmed directly (`az account list-locations`) that **Azure has no Nigeria region at all** — only
South Africa North/West. So "redeploy to Azure-in-Nigeria" is not an option that exists today.

The owner has a specific lead to check: **Huawei Cloud**, which the owner believes has a real presence in
Nigeria, and a possible (unconfirmed, personal-connection-dependent) route to cloud credits there. Verify
this directly rather than take it on faith — confirm whether Huawei Cloud actually operates a real
availability zone/region physically in Nigeria today, not just a sales/support presence, and what it
would actually take to run a production PostgreSQL database and a containerized ASP.NET Core API there.

## What to research

**1. Does Huawei Cloud actually have a real, operational presence in Nigeria?** Find primary sources
(Huawei Cloud's own region/availability-zone documentation, press releases with dates, Nigerian
tech-press coverage) confirming or denying this. If it exists:
   - What compute/database services does it offer there specifically — is there a managed PostgreSQL
     equivalent (like Azure's Flexible Server), or would GARBATI need to self-manage Postgres on a VM
     (ECS)? Self-managing Postgres is a real operational step up from today's fully-managed Flexible
     Server (backups, patching, HA all become GARBATI's own job) — say so plainly if that's the case.
   - Does it support point-in-time recovery or an equivalent backup mechanism comparable to what
     `LedgerApi/docs/DISASTER-RECOVERY.md` already documents and tested against Azure?
   - Rough, realistic pricing for a small instance comparable to what's running today (`Standard_B1ms`,
     32 GiB storage) — order of magnitude is fine, not an exact quote.
   - Does Huawei Cloud actually offer a startup/credit program a solo developer could realistically access
     in Nigeria, and what would that path look like (apply directly, via a partner, etc.)? **Do not assume
     the owner's personal-connection lead will work out** — research the standard, public path that exists
     regardless of that connection, and note the connection as a separate, unverifiable-by-you factor.

**2. What other real hosting options exist physically inside Nigeria?** Survey the actual Nigerian data
center / cloud market, not just Huawei — e.g. (verify each, don't assume the name alone means it's real
and current):
   - Rack Centre
   - MainOne / Equinix (Equinix acquired MainOne)
   - Open Access Data Centres (OADC)
   - Africa Data Centres (Liquid Intelligent Technologies)
   - Galaxy Backbone (government-owned)
   - Any others a real Nigerian tech-infrastructure search surfaces
   For each one found to be real and current, note: is it colocation-only (GARBATI would need to supply
   and manage its own servers/VMs) or does it offer a managed cloud/database layer closer to what Azure
   provides today? This matters a lot for effort — colocation is a much bigger operational lift for a
   solo developer than a managed PaaS database.

**3. Try again, harder, for the primary CBN circular text, and confirm actual scope.** T-037 and
`research/nigerian-regulatory-requirements.md` both already hit a wall trying to fetch primary CBN PDFs
directly (403s/image-only). Try different approaches (CBN's own website structure, cached/archived
versions, Nigerian legal databases, law-firm client alerts that quote the circular's actual text rather
than summarize it) to answer specifically:
   - Is "microfinance bank" or "MFI" named anywhere in the actual circular's coverage, or is coverage
     genuinely limited to the categories already found ("banks, payment service providers, fintech
     companies")?
   - Does the circular distinguish "storage" from "processing" in a way that might allow a hybrid
     approach (e.g., a Nigeria-based read replica or backup copy while primary processing stays
     elsewhere), or is it unambiguous that both must be in-country?
   - Is there any stated exemption, grace period, or phased-compliance path for smaller institutions?
   If the primary text still can't be reached, say so plainly (as both prior briefs did) rather than
   stretch a secondary summary further than it supports.

**4. Honest effort/risk comparison.** Given what's found, lay out — as a comparison, not a recommendation
you make for the owner — roughly what each realistic path costs in effort and risk:
   - Stay on Azure South Africa North, accept the compliance risk, revisit once real customers exist.
   - Migrate LedgerApi's database only to a Nigerian provider (keep the other 4 services + App Service
     hosting on Azure, since only "payment and financial transaction data" is named — note if this
     split-hosting approach seems legally coherent per what you find, or if that's itself an open question
     needing compliance confirmation).
   - Migrate the whole platform to a Nigerian provider.
   State clearly that the actual decision (which path, and when) is the owner's, informed by whatever is
   confirmed here plus real compliance-officer sign-off — not something this research resolves on its own.

## Output

Write the brief to `research/nigeria-hosting-options.md`, matching this repo's existing research-brief
style (cited sources with dates where possible, clear findings, an explicit "what I could not verify"
section). Read `research/oss-core-banking-readiness-gaps.md`'s G4 section and
`research/nigerian-regulatory-requirements.md` first for context — don't repeat their already-documented
dead ends without at least trying a different angle first.

## Out of scope

- Actually migrating anything, or writing any infrastructure/deployment code.
- Deciding which hosting path GARBATI takes — that's the owner's call once this brief exists.
- Re-litigating G3 (BVN/NIN encryption) or G7 (secrets management) — related, but separate tasks.
