# Research: real Nigeria-based hosting options for LedgerApi's database

Task: T-041 (follow-up to T-037/G4). **This document does not decide anything and does not recommend a
single path.** It lays out what real hosting options physically inside Nigeria actually exist today, what
each would really cost in effort, and what's still genuinely uncertain about the CBN circular driving this
— for the owner to weigh, with compliance-officer sign-off, before any migration is written. No code was
written or changed as part of this research.

## Plain-language summary

The short version: Azure has no Nigeria region (confirmed directly by the coordinator before this task
started — `az account list-locations` shows only South Africa North/West), and neither does Google Cloud
(confirmed in this research — nearest GCP region is also Johannesburg, with only CDN/edge/interconnect
points of presence in Lagos). **AWS is the one major hyperscaler with an actual physical presence inside
Nigeria** — a Local Zone in Lagos, live since January 2023 — but it only confirms basic compute (EC2) there,
not a managed database; whether RDS (AWS's managed Postgres) runs in that specific Local Zone could not be
confirmed either way. Outside the big three hyperscalers, **Huawei Cloud is real and does have a genuine,
growing, several-years-old presence in Nigeria** (not just a sales office) — but what's actually confirmed
to run there is compute and storage, not a confirmed managed-PostgreSQL-equivalent; the generic global
Huawei startup-credit program explicitly excludes Nigeria (it's restricted to Singapore/South Africa
companies with enterprise certification), and the owner's personal-connection credit lead is a separate,
unverifiable-by-me factor on top of that. Nigeria's own data-centre market (Rack Centre, MainOne/Equinix,
Open Access Data Centres, Africa Data Centres, Galaxy Backbone) is large and genuinely growing fast, but
**every option found is colocation/IaaS-first** — none was confirmed to offer a drop-in managed-PostgreSQL
equivalent to what Azure Flexible Server already gives GARBATI for free today. **Galaxy Backbone is the one
provider actively marketing itself specifically to banks and fintechs because of this exact CBN circular**,
which makes it worth a direct conversation, but its pricing and solo-developer accessibility are opaque
(phone-only contact found, no public pricing page).

On the regulatory question: I found the actual CBN PDF's real URL on cbn.gov.ng this time — a specific,
new result this research adds — but fetching it still returned `403 Forbidden`, the same wall every prior
research effort has hit. What changed is the secondary-source picture: **six independent sources (five
separate Nigerian/pan-African law firms plus one financial-news outlet) now consistently and explicitly
name "microfinance banks" in the circular's addressee list**, which is a real improvement over T-037's "I
could not confirm" — I'm treating this as medium-high confidence now, not primary-source-verified. The
same sources are also consistent that **there is no stated exemption or phased-compliance path for smaller
institutions, and the circular does not appear to allow a storage/processing split** — one law firm's
client-alert states plainly that "access is itself a form of processing: a foreign-based team with login
access to your Nigerian payment data constitutes a cross-border transfer, regardless of where the servers
sit." If that reading holds, it has a real, non-obvious consequence for GARBATI's "migrate only LedgerApi's
database" idea: if OrchestratorApi (which would stay on Azure) keeps reading and writing LedgerApi's
database as part of normal operation, that cross-border access might itself be "processing" outside Nigeria
under this reading — which would undercut the split-hosting approach's compliance logic. This is my own
inference connecting the circular's described scope to GARBATI's own architecture, not something any source
states about GARBATI specifically — **it needs direct compliance-officer confirmation before anyone treats
split-hosting as a safe assumption.**

Confidence overall: **high** for what's confirmed to not exist (Azure/GCP have no Nigeria region — directly
checked); **medium-high** for Huawei Cloud's and AWS's Nigeria presence existing and roughly what it offers
(multiple independent primary-ish sources: vendor's own press releases/docs, tech press, one official AWS
docs page); **medium** for exactly which managed-database services are actually available at each Nigerian
location (the single weakest-verified technical question in this brief — flagged explicitly below);
**medium-high** (newly improved from T-037's "unconfirmed") that microfinance banks are named in the CBN
circular's scope, still secondary-source-only; **low** on whether a storage/processing split
("split-hosting") would actually satisfy the circular, which is my own reading of secondary sources, not a
compliance opinion.

---

## 1. Huawei Cloud in Nigeria — is it real, and what does it actually offer?

**Yes, it's real — not just a sales presence.** Huawei has made three separate, dated, concrete
announcements about Nigeria infrastructure, not one vague claim:

- **December 10, 2024** — launch event in Lagos for what Huawei calls Nigeria's "first hyperscale local
  cloud," described as Tier 3+ data centre infrastructure with ~15ms latency, alongside a three-year plan
  (1,000 customer adoptions, 100 partners, 5,000 developers trained) and two training centres (Lagos,
  Abuja) ([Huawei Cloud's own announcement](https://www.huaweicloud.com/intl/en-us/news/20241212084526769.html)).
- **~October 31, 2024** (an earlier soft-launch date per trade press) — described explicitly as "an
  extension of [Huawei's] existing cloud infrastructure in Johannesburg, South Africa," also linked to
  Dublin, Ireland infrastructure ([ITWeb Africa](https://itweb.africa/content/Olx4zMka3PXv56km)). This is
  an important nuance: Nigeria is not described as a fully independent Huawei Cloud region with its own
  region code in the way Johannesburg is — it reads more like a local point of presence/availability zone
  layered on the Johannesburg region's control plane and (likely) service catalog, though I could not
  directly confirm the exact technical architecture from Huawei's own documentation.
- **August 27–28, 2026** — Huawei launched an "Agentic AI Cloud" in Nigeria at a Lagos summit, explicitly
  described as building on "earlier deployments of a hyperscale local cloud in 2024 and local data-cloud
  services in 2025" ([TechAfrica News](https://techafricanews.com/2026/08/31/huawei-launches-agentic-ai-cloud-nigeria/),
  [Techpoint Africa](https://techpoint.africa/brandpress/huawei-deepens-local-infrastructure-push-with-agentic-ai-cloud-launch-in-nigeria/)).

**What's confirmed to run there vs. what isn't.** Huawei's own Nigeria compliance page confirms general
service purchase is possible ("Individuals, enterprises, and organizations can register local accounts on
the Huawei Cloud International website and purchase cloud services" —
[Huawei Cloud Nigeria compliance centre](https://www.huaweicloud.com/intl/en-us/securecenter/compliance/compliance-center/ng.html)),
and general coverage describes "data storage and computing resources" and "cloud-native databases" in
Nigeria specifically. But I could **not** confirm that Huawei's managed-PostgreSQL product, **RDS for
PostgreSQL, is actually offered as a Nigeria-local service** — every specific regional-availability
reference I found for RDS for PostgreSQL in Africa names only the **AF-Johannesburg** region, not a
separate Nigeria region. **If RDS for PostgreSQL is not actually available Nigeria-locally, GARBATI would
need to self-manage PostgreSQL on a Huawei Cloud ECS (compute) instance instead — which is a real
operational step up from today's fully-managed Azure Flexible Server: backups, patching, version upgrades,
and high availability would all become GARBATI's own job, not Huawei's, exactly as the task described this
risk.** This is the single most important unresolved technical fact about the whole Huawei option and
needs direct confirmation from Huawei's own Nigeria sales/support channel (or the Huawei Cloud console's
region selector, if the owner has an account) before anyone treats "Huawei Cloud in Nigeria" as a like-for-
like replacement for what Azure Flexible Server already does.

**Point-in-time recovery.** Huawei's RDS for PostgreSQL product (where it is available) does document
automated backups and restore capability in its general product docs, broadly comparable in concept to
Azure Flexible Server's PITR — but since I could not confirm RDS for PostgreSQL runs in Nigeria specifically
(see above), I cannot say this backup/PITR capability is actually available there. If GARBATI ends up
self-managing Postgres on an ECS instance instead, there is no managed PITR equivalent — GARBATI would need
to build and test its own backup/WAL-archiving mechanism, which is a materially larger task than anything
`LedgerApi/docs/DISASTER-RECOVERY.md` currently documents (that document assumes a managed service's PITR
as the starting point, not something built from scratch).

**Rough pricing, order of magnitude only.** A small Huawei Cloud ECS instance (`t7.large.1`, 2 vCPU/2 GiB)
priced for the AF-Johannesburg region came back at roughly **$6.60/month** for compute alone
([mofcloud.com pricing aggregator](https://mofcloud.com/tools/servers/HuaweiCloudIntl/t7.large.1/)) — for
comparison, Azure's `Standard_B1ms` (1 vCPU/2 GiB, the SKU `LedgerApi` actually runs today) prices at
roughly **$16/month** for compute alone in South Africa North
([bytebase.com dbcost aggregator](https://www.bytebase.com/dbcost/azure-flexible/instance/B1ms/)). These
are not directly comparable SKUs and neither figure includes storage, backup, or networking, and I could
not find Nigeria-specific pricing (only Johannesburg) — treat this only as "same general order of
magnitude, possibly somewhat cheaper for raw compute," not a real quote. A real quote requires Huawei's own
pricing calculator with a Nigeria-specific region selected, which I did not have account access to use.

**Startup credits — the owner's lead, verified honestly.** Two separate Huawei credit programs exist, and
neither matches what the owner described without caveats:

1. **Huawei Cloud's generic global "Start Up" credit program** (the public page a solo developer would find
   by default) **explicitly excludes Nigeria**: it states the plan "is open to enterprise users only"
   (requires completed enterprise certification — not available to an individual/sole-proprietor account)
   and "currently is available only to Singapore and South African companies"
   ([Huawei Cloud Start Up program page](https://www.huaweicloud.com/intl/en-us/activity/start-up/index0.html)).
   This is a direct, concrete finding: **the standard public path does not work for a Nigerian solo
   developer today**, regardless of any personal connection.
2. **A separate, Nigeria-specific program announced April 2024**: $10 million in cloud credits across 100
   Nigerian startups, up to $150,000 per startup, launched at a Huawei Cloud Startup Summit in Lagos
   ([TechCabal](https://techcabal.com/2024/04/06/huawei-cloud-to-empower-100-nigerian-startups-to-boost-cloud-market/),
   [Dabafinance](https://dabafinance.com/en/news/chinese-giant-huawei-to-offer-10m-cloud-credits-to-nigerian-startups)).
   I could not confirm this program's application process, eligibility (does it require a registered company
   vs. an individual developer?), or whether it is still accepting applications as of October 2026 — the
   TechCabal article itself returned a direct-fetch error when I tried to read it for process details, so
   this is reported from search-result summaries only, not the article's full text.

**The owner's personal-connection lead specifically:** I'm treating this exactly as the task asked — as a
real, separate, unverifiable-by-me factor, not something to assume will work. Nothing above should be read
as either confirming or ruling out that connection; it is simply outside what research can check. The
standard public paths, independent of that connection, are: (a) not available for Nigeria via the generic
global program, and (b) possibly available via the Nigeria-specific 2024 program, but with real unknowns
about current eligibility and process that only a direct inquiry to Huawei can resolve.

---

## 2. Other real hosting options physically inside Nigeria

All six names from the task, plus AWS's Local Zone (found during this research and directly relevant,
since it's the one major hyperscaler with real Nigeria presence), checked for whether each is real/current
and whether it's colocation-only or offers something closer to managed cloud/database.

| Provider | Real & current? | Colocation-only, or managed cloud/DB? | What I actually found |
|---|---|---|---|
| **Rack Centre** | Yes — largest colocation operator in Nigeria, Lagos, two facilities including a new 12MW hyperscale-ready campus (LGS2) | **Both.** Core business is carrier-neutral colocation, but it also runs **CloudOnGround™**, a multi-provider marketplace reportedly offering IaaS/PaaS/SaaS/DRaaS from "multiple local and international cloud providers," plus an official AWS Direct Connect location. I could not identify which specific providers/services are live on CloudOnGround™ today or whether a managed PostgreSQL-equivalent is among them. | ISO 9001/27001/22301/45001, PCI DSS, SOC 1 Type II certified ([rack-centre.com](https://rack-centre.com/), [Rack Centre Cloud Services](https://rack-centre.com/data-centre-services/cloud/)) |
| **MainOne (now "Solutions by Equinix")** | Yes — Equinix acquired MainOne in 2022; actively expanding (a new $22M "LG3" facility targeted for Q1 2026, an "LG2.3" expansion already opened) | **Managed Private Cloud is VMware-based IaaS** — "Managed Private Cloud (MPC) powered by VMware... brings Infrastructure-as-a-Service to West African businesses." This reads as **self-managed VMs**, not a managed-database layer; I could not confirm a managed PostgreSQL product exists on this platform (the specific product page returned a fetch error, so this is from search-result summaries only). | [TechCabal on the $22M LG3 build](https://techcabal.com/2025/11/10/equinix-to-build-22-million-data-centre-in-lagos/), [MainOne Managed Private Cloud page](https://mainone.net/mainone-cloud-solutions/managed-private-cloud/) (page itself returned 403 on direct fetch) |
| **Open Access Data Centres (OADC)** | Yes — Lekki, Lagos flagship facility live; $240M/24MW expansion announced, targeted for 2027; backed by WIOCC, IFC, Proparco, ACA | **Colocation + connectivity fabric**, not a confirmed managed-database layer. OADC's "Open Access Fabric" lets customers connect at 1G/10G/100G to Azure, Google Cloud, and AWS — this is a **network on-ramp to those hyperscalers' real regions elsewhere**, not those hyperscalers' compute/storage physically landing in Nigeria. A newer partnership with "UniCloud" announced a "sovereign cloud" offering (local currency billing, zero egress fees, ISO-certified in-country data management) but is explicitly GPU/AI-focused (GPUaaS) in what I found, with no managed relational-database product mentioned. | [Businessday on OADC's Open Access Fabric](https://businessday.ng/technology/article/oadcs-open-access-fabric-to-unlock-cloud-access-for-nigerias-200m-population/), [Businessday on the UniCloud/OADC sovereign cloud deal](https://businessday.ng/technology/article/africa-moves-to-end-data-colonialism-as-unicloud-oadc-seal-sovereign-cloud-deal/), [Leadership.ng on the $240M expansion](https://leadership.ng/oadc-announces-240m-investment-in-24mw-lagos-data-centre/) |
| **Africa Data Centres (Liquid Intelligent Technologies)** | Yes — building a new Lagos facility; part of the pan-African Liquid group | **Colocation plus resold hyperscaler access.** Liquid's own description is "connectivity, cloud, cybersecurity, managed solutions... combines owned infrastructure with resold/managed hyperscaler products including Microsoft Azure, Google Cloud." **Important distinction:** "resold" strongly suggests Liquid is reselling access to Azure's/GCP's real regions (which are not in Nigeria), not hosting Azure/GCP compute physically inside Nigeria — this would **not** satisfy a hard in-country-storage requirement on its own. This needs direct confirmation before assuming otherwise. | [Liquid Intelligent Technologies data centres page](https://liquid.tech/data-centres/), [tech.africa on the new Lagos facility](https://tech.africa/africa-data-centres-lagos/) |
| **Galaxy Backbone** | Yes — federal-government-owned, established 2006, Tier III National Shared Services Centre (Abuja) plus a Tier IV disaster-recovery site (Kano) | **Both, and most directly positioned for this exact use case.** Runs "1Government Cloud" / "Galaxy Backbone Cloud Platform (GxCP)," described as a "Government Private Cloud" and, in recent coverage, a "sovereign cloud platform," PCI DSS certified. **As of July 2026, Galaxy Backbone is actively, publicly marketing itself to banks and fintechs specifically because of this CBN data-localisation circular** — this is the one provider found in this research with a stated go-to-market pitch aimed exactly at GARBATI's situation. I could **not** find a public pricing page, self-serve signup flow, or confirmation of whether a solo developer/small startup (versus a bank or large fintech) is a realistic customer — the only contact method found was a phone number. | [Vanguard: "CBN Data Localisation: Galaxy Backbone moves to capture banks, fintechs"](https://www.vanguardngr.com/2026/07/cbn-data-localisation-galaxy-backbone-moves-to-capture-banks-fintechs/), [Galaxy Backbone services page](https://galaxybackbone.com.ng/services/) |
| **AWS Local Zone, Lagos** (not in the task's original list — found during this research) | Yes — live since January 2023, the only AWS Local Zone in Africa, parented to the `af-south-1` (Cape Town) region, zone ID `af-south-1-los-1a` | **EC2 (compute) confirmed available; managed RDS PostgreSQL NOT confirmed available in this specific Local Zone.** AWS Local Zones generally offer only a subset of their parent region's services, and RDS support across Local Zones worldwide is documented as rare (one independent source names only the Los Angeles Local Zone as confirmed to support RDS) — I found no direct confirmation either way for Lagos specifically. If RDS is not available, this has the same "self-managed Postgres on a VM" operational profile as the Huawei option. | [AWS's own Local Zones list, confirming the Lagos zone exists](https://docs.aws.amazon.com/local-zones/latest/ug/available-local-zones.html), [Connecting Africa on the Lagos launch](https://www.connectingafrica.com/cloud-networking/lagos-launches-first-aws-local-zone-in-africa) |

**For completeness, the two hyperscalers already ruled out:** Azure confirmed to have no Nigeria region
(coordinator's direct check, prior to this task). **Google Cloud also confirmed to have no Nigeria region**
in this research — nearest GCP region is Johannesburg; Nigeria only has GCP CDN/edge and Dedicated Cloud
Interconnect points of presence, not compute/database infrastructure
([search-result summary citing GCP's own edge-locations documentation](https://docs.cloud.google.com/vpc/docs/edge-locations)).
This means the "Azure has no Nigeria region" gap is not Azure-specific — it's true of all three major
Western hyperscalers; only AWS has any physical Nigeria footprint at all, and even that is compute-only as
far as this research could confirm.

**The pattern worth naming plainly:** across every option surveyed, **a genuinely managed, Flexible-Server-
equivalent PostgreSQL service physically running inside Nigeria was not confirmed to exist anywhere** — not
at Huawei, not at AWS's Lagos Local Zone, not at any of the five Nigerian data-centre operators. The
realistic baseline assumption for any Nigeria-hosting path is **self-managed PostgreSQL on a VM**, which
means GARBATI would newly own backup scheduling, patching, version upgrades, and high-availability — all of
which Azure Flexible Server currently does automatically. This is the single biggest hidden cost in every
option above, and it's consistent across providers rather than specific to any one of them.

---

## 3. The CBN circular — trying again for the primary text, and what changed

### What's new in this attempt

I found the circular's actual file on cbn.gov.ng's own domain this time, via a search result rather than a
direct guess at the URL pattern:

```
https://www.cbn.gov.ng/Out/2026/CCD/CIRCULAR ON INTRODUCTION OF MARKET STRUCTURE REQUIREMENTS, DATA
LOCALISATION, ULTIMATE BENEFICIAL OWNERSHIP DISCLOSURE, AND SYSTEMIC OVERSIGHT MEASURES IN THE NIGERIA
PAYMENTS SYSTEM.pdf
```

Fetching it directly still returned **`403 Forbidden`** — the exact same wall `research/nigerian-regulatory-
requirements.md` and T-037 both already documented for other CBN PDFs. **This is a real, confirmed dead
end, not a new success** — I'm naming the exact URL here specifically so a future attempt (e.g., from a
Nigeria-based IP address, via CBN's FOI contact, or by directly emailing the Payments System Supervision
Department) has a concrete target rather than having to search for it again.

### What the secondary-source picture now shows (stronger than T-037)

T-037 could not confirm whether microfinance banks were named in the circular's scope. This research found
**six independent secondary sources** — five separate Nigerian/pan-African law firms and one aggregated
financial-news summary — that now **consistently and explicitly list "microfinance banks"** among the
circular's named addressees:

> "...deposit money banks, microfinance banks, mobile money operators, switching and processing companies,
> payment terminal and solution service providers, super agents, and all other licensed payment
> operators." — consistently reported, in near-identical wording, by
> [Aluko & Oyebode](https://www.aluko-oyebode.com/insights/cbn-issues-circular-on-market-structure-data-localisation-and-systemic-oversight-in-nigeria-payments/),
> [Enebeli and Partners Legal](https://goenebeli.com/cbn-circular-pss-dir-pub-cir-001-004-15-june-2026new-rules-on-data-localisation-beneficial-ownership-disclosure-market-concentration-in-the-nigerian-digital-payments-ecosystem/),
> and **Kiet Business Attorneys' client-alert PDF**, which I read in full (see below).

Two other sources (Olaniwun Ajayi LP, a well-known Nigerian commercial law firm, and allAfrica's syndication
of a Vanguard report) described the scope more generically as "Deposit Money Banks, Payment Service
Providers and other financial institutions" **without** naming microfinance banks specifically — but
neither of these sources stated microfinance banks are *excluded* either; they simply used a shorter
summary phrase. This is not a direct contradiction, more a difference in how thoroughly each firm's summary
quoted the addressee list. Given that the fuller, more specific quotations are the ones that name
microfinance banks explicitly and consistently, **I'm raising confidence on this point to medium-high** —
still secondary-source-only, not primary-text-verified, but no longer "could not confirm" the way T-037 had
to leave it.

**The most substantive new document found:** a five-page client-alert PDF from Kiet Business Attorneys
(dated June 2026, titled "The CBN Payments System Circular"), which I read in full via direct PDF
extraction rather than just a web-fetch summary. It is explicitly **a law firm's own summary document, not
the primary CBN circular itself** — it says so on its final page ("This publication is a general legal
overview and does not constitute legal advice") — but it is the most detailed and specific secondary source
found, and states several things plainly that matter for GARBATI:

- **Scope is wide, by data type, not just institution type:** "Transaction records, BVN data, card data,
  settlement records, payment instructions, authentication credentials, associated metadata, and the
  systems that consume this data; fraud models, analytics pipelines, risk-scoring engines are all covered."
- **No storage/processing split appears to be permitted:** "Critically, access is itself a form of
  processing: a foreign-based team with login access to your Nigerian payment data constitutes a
  cross-border transfer, regardless of where the servers sit." And separately, from a different source
  (Mondaq): "This extends beyond maintaining a local copy of data and requires that the primary processing
  environment, databases, backups and operational control remain on infrastructure located within Nigeria"
  ([Mondaq](https://www.mondaq.com/nigeria/financial-services/1812264/cbns-data-localisation-directive-compliance-considerations-for-payment-system-participants)).
- **No exemption or phased-compliance path for smaller institutions was found in any source** — every
  source consistent on a flat **January 1, 2027** deadline for data localisation, applying uniformly to
  everyone named in scope (a separate, earlier **December 31, 2026** deadline applies only to a different
  obligation in the same circular — monthly market-share-cap reporting — not to data localisation itself).
- **The circular is dated 15 June 2026**, reference **PSS/DIR/PUB/CIR/001/004**, signed by **Dr Rakiya O.
  Yusuf, Director, Payments System Supervision Department** — this detail is now confirmed consistently
  across every source found, including the one PDF I read directly.
- **Enforcement is stated as real, not theoretical:** the Kiet brief notes "recent enforcement actions
  against fintech platforms" as evidence the CBN "acts publicly and at scale" on circulars like this one — I
  did not independently verify which specific enforcement actions are being referenced, so treat this as the
  law firm's framing, not an independently confirmed fact.

### The open legal question this raises for GARBATI specifically — not resolved here

If "access is itself a form of processing" is an accurate reading of the circular (again: from a law firm's
summary, not primary text I read myself), it has a direct, concrete implication for the "migrate only
LedgerApi's database" option the task asked me to compare: **GARBATI's own architecture (`ARCHITECTURE.md`
P-5) has OrchestratorApi call LedgerApi's API to post transactions — not a shared database, so this
specific "foreign team with login access to the database" scenario the law firm describes may not directly
apply the same way an ops engineer manually querying the database would.** But if LedgerApi's *application
tier* (not just its database) also needs to run inside Nigeria for this reasoning to be satisfied — because
the application process itself is "processing" the data, and it would otherwise run on Azure outside Nigeria
even with a Nigeria-hosted database — that's a materially different, larger migration than "database only."
**This is exactly the kind of reading only a qualified Nigerian compliance officer (ideally one with direct
payments-regulation experience) can resolve with confidence — I am flagging it as a real, concrete
distinction the owner needs confirmed, not answering it myself.**

---

## 4. Honest effort/risk comparison of the three paths

As the task asked: **a comparison, not a recommendation.** The actual decision is the owner's, informed by
this research plus real compliance-officer sign-off.

### Path A — Stay on Azure South Africa North, accept the compliance risk, revisit once real customers exist

- **Effort now:** zero — nothing changes.
- **Risk:** real, dated, and not merely theoretical. The CBN circular (per six consistent secondary
  sources) sets a flat 1 January 2027 deadline with no stated exemption for smaller institutions, and one
  source describes CBN as already taking visible enforcement action against fintech platforms on related
  matters. **The concrete mitigating factor, already noted in `ARCHITECTURE.md` Section 9, is that GARBATI
  has no real customer data in production yet** — so the actual exposure today is close to zero, and grows
  only once real customer payment data exists on Azure past the deadline.
- **What would make this riskier fast:** onboarding any real, paying customer whose payment data would be
  "Nigeria-originated" before a hosting decision is made and acted on.

### Path B — Migrate only LedgerApi's database to a Nigerian provider, keep the other four services + App
Service hosting on Azure

- **Legal coherence: genuinely uncertain, not just unconfirmed.** The circular's stated scope (per secondary
  sources) is "payment transaction data," and LedgerApi is the one GARBATI service that actually holds that
  data by the platform's own design (P-5: database-per-service, no shared DB). On its face this looks like
  the most surgical, lowest-effort compliant option. But Section 3 above raises a real, unresolved question:
  if "processing," not just "storage," must also happen inside Nigeria, and if LedgerApi's own application
  tier (currently Azure App Service) counts as part of that processing, then database-only migration might
  not actually satisfy the circular even though it moves the data itself into Nigeria. **This needs explicit
  compliance-officer confirmation before anyone treats it as sufficient.**
- **Effort if it is sufficient:** meaningfully smaller than a full platform migration — one database
  (currently Standard_B1ms / 32 GiB) needs a new home, with the operational step-up described in Section 2
  (self-managed Postgres almost certainly, given no confirmed managed-Postgres-equivalent was found at any
  surveyed Nigerian provider) — backup/PITR, patching, and HA all become GARBATI's own responsibility,
  replacing what Azure Flexible Server currently does automatically per `LedgerApi/docs/DISASTER-RECOVERY.md`.
- **Effort if the application tier must move too:** LedgerApi's App Service would also need to move to
  whichever Nigerian compute option is chosen (self-managed containers on a VM in every case surveyed — no
  Nigerian option was confirmed to offer an Azure-App-Service-equivalent managed container/PaaS runtime
  either), which is a real deployment-pipeline change (CI/CD target, secrets, networking) on top of the
  database move.

### Path C — Migrate the whole platform to a Nigerian provider

- **Effort:** the largest of the three by a clear margin — every one of GARBATI's five services (LedgerApi,
  Garbati.IdentityApi, Garbati.OrchestratorApi, Garbati.StaffIdentityApi, Garbati.OpsApi) would need a new
  home, and since no Nigerian option surveyed was confirmed to offer a managed PaaS-plus-managed-Postgres
  combination equivalent to what Azure currently provides for free, this almost certainly means standing up
  and operating self-managed infrastructure (VMs, containers, backups, patching, HA) across the whole
  platform — a genuinely large operational step up for a solo developer, not a configuration change.
- **Compliance clarity:** cleanest of the three — avoids the "does cross-border access to a Nigeria-hosted
  database still count as processing outside Nigeria" ambiguity entirely, since nothing would be left on
  Azure to raise that question. But "cleanest compliance story" and "realistic for a solo developer to
  operate well" are in direct tension here, and that tradeoff is the owner's to weigh, not mine to resolve.
- **Cost:** the one piece of good news found in this research — rough per-instance compute pricing at both
  Huawei (Johannesburg reference pricing) and Azure (today's actual South Africa North pricing) came back in
  the same general order of magnitude (roughly $7–20/month for small compute instances), so a full migration
  is not obviously more expensive in raw infrastructure cost — the real cost is the operational effort of
  self-managing what's currently fully managed, not a steep rise in the cloud bill itself.

---

## What I could not verify

- **Whether Huawei Cloud's managed RDS for PostgreSQL service actually runs in Nigeria specifically**, as
  opposed to only in the AF-Johannesburg region with Nigeria offering compute/storage/other services. This
  is the single most consequential unresolved technical fact in this brief — it determines whether the
  Huawei option looks like "today, but in Nigeria" (managed DB) or "a real new operational burden"
  (self-managed Postgres on a VM). Needs direct confirmation from Huawei's own sales/support channel or a
  console region-selector check.
- **Whether AWS RDS (managed PostgreSQL) is available specifically in the Lagos Local Zone**, as opposed to
  only EC2 compute there. AWS's own documentation confirms the Lagos Local Zone exists and confirms EC2 is
  available; I found no source confirming or denying RDS availability there specifically (one source notes
  RDS-in-Local-Zones support is rare worldwide, naming only a Los Angeles example as confirmed).
- **Whether any of the five Nigerian data-centre operators surveyed (Rack Centre, MainOne/Equinix, OADC,
  Africa Data Centres, Galaxy Backbone) actually offers a managed relational-database product** comparable
  to Azure Flexible Server, versus colocation/IaaS requiring GARBATI to self-manage Postgres. Every provider
  page I could fetch described infrastructure/connectivity/IaaS in general terms; none I could confirm
  explicitly named a managed-Postgres-equivalent product. Two provider pages (MainOne's Managed Private
  Cloud page, TechCabal's Huawei-startup-program article) returned direct fetch errors (403/500), so those
  specific sources are reported from search-result summaries only, not full-text reads.
- **The primary CBN circular text itself.** I found and can report the exact URL
  (`https://www.cbn.gov.ng/Out/2026/CCD/CIRCULAR ON INTRODUCTION OF...pdf`) but could not fetch it — same
  `403 Forbidden` wall as every prior attempt in this repo's research. Everything about the circular's exact
  wording in this brief comes from secondary legal-commentary sources, including one I read in full (a law
  firm's own client-alert PDF, not the primary document), not the primary text.
- **Whether microfinance banks are definitively named in the circular** — upgraded to medium-high confidence
  (six consistent independent secondary sources now name them explicitly) from T-037's "could not confirm,"
  but still not primary-source-verified.
- **Whether a storage/processing split (e.g., a Nigeria-based replica while primary processing stays on
  Azure) would satisfy the circular.** Secondary sources lean toward "no" (one explicitly states remote
  access itself counts as cross-border processing), but this is a reading of secondary commentary, not a
  primary-text-confirmed legal conclusion, and I could not find a source addressing GARBATI's specific
  architecture (API-only cross-service calls, no shared database) directly.
- **The Nigeria-specific Huawei $10M/100-startups credit program's current (October 2026) eligibility and
  application process** — found only in April 2024 announcement coverage; could not confirm whether it is
  still open, what documentation it requires, or whether an individual/sole-proprietor qualifies.
- **Exact current pricing for any Nigerian hosting option** — every pricing figure in this brief is either
  a reference price from a different region (Johannesburg, for Huawei) or absent entirely (Galaxy Backbone,
  OADC, MainOne, Rack Centre all lacked public pricing pages in what I could find) — all figures here are
  order-of-magnitude estimates, not quotes.
- **The owner's personal-connection lead to Huawei Cloud credits** — by design, outside what research can
  verify; noted as a separate, unconfirmed factor per the task's own instruction.

## What this affects

This directly informs the open question already logged in `ARCHITECTURE.md` Section 9, **"CBN
data-localisation deadline"** (added 22 Oct 2026 per T-037/G4, which named this exact follow-up task,
T-041, as in progress). That entry should be updated to reflect: (1) a specific, confirmed-dead primary-CBN-
PDF URL for any future retry attempt; (2) improved (medium-high, still secondary-source) confidence that
microfinance banks are named in the circular's scope; (3) the real, unresolved "does cross-border access to
a Nigeria-hosted database count as processing outside Nigeria" question that bears directly on whether
"migrate LedgerApi's database only" is actually sufficient; and (4) the concrete finding that no Nigerian
hosting option surveyed — including Huawei Cloud — was confirmed to offer a managed-PostgreSQL-equivalent
to Azure Flexible Server, meaning every realistic path involves GARBATI taking on self-managed-database
operations it does not currently have to do. The actual choice among Path A/B/C above, and its timing
relative to the 1 January 2027 deadline, remains entirely the owner's decision, to be made with a qualified
Nigerian compliance officer's direct input — not something this research resolves.

## Sources

**Huawei Cloud Nigeria:**
- [Huawei Cloud — "Huawei Launches Nigeria's First Hyperscale Local Cloud" (own press release, 10 Dec 2024)](https://www.huaweicloud.com/intl/en-us/news/20241212084526769.html)
- [ITWeb Africa — "Huawei to open cloud region in Nigeria by end October" (2024)](https://itweb.africa/content/Olx4zMka3PXv56km)
- [TechAfrica News — "Huawei Launches Agentic AI Cloud in Nigeria" (31 Aug 2026)](https://techafricanews.com/2026/08/31/huawei-launches-agentic-ai-cloud-nigeria/)
- [Techpoint Africa — "Huawei Deepens Local Infrastructure Push with Agentic AI Cloud Launch in Nigeria"](https://techpoint.africa/brandpress/huawei-deepens-local-infrastructure-push-with-agentic-ai-cloud-launch-in-nigeria/)
- [Huawei Cloud — Nigeria compliance/data-protection centre page](https://www.huaweicloud.com/intl/en-us/securecenter/compliance/compliance-center/ng.html)
- [Huawei Cloud — generic global Start Up credit program page (confirms Nigeria exclusion)](https://www.huaweicloud.com/intl/en-us/activity/start-up/index0.html)
- [TechCabal — "Huawei Cloud to empower 100 Nigerian startups" ($10M program, Apr 2024)](https://techcabal.com/2024/04/06/huawei-cloud-to-empower-100-nigerian-startups-to-boost-cloud-market/)
- [Dabafinance — "Chinese giant Huawei to offer $10m cloud credits to Nigerian startups"](https://dabafinance.com/en/news/chinese-giant-huawei-to-offer-10m-cloud-credits-to-nigerian-startups)
- [mofcloud.com — Huawei Cloud `t7.large.1` Johannesburg reference pricing](https://mofcloud.com/tools/servers/HuaweiCloudIntl/t7.large.1/)
- [bytebase.com dbcost — Azure Flexible Server `B1ms` South Africa North reference pricing](https://www.bytebase.com/dbcost/azure-flexible/instance/B1ms/)

**Other Nigerian/African hosting options:**
- [Rack Centre — company site](https://rack-centre.com/), [Cloud Services (CloudOnGround™) page](https://rack-centre.com/data-centre-services/cloud/)
- [TechCabal — "Equinix to build $22 million data centre in Lagos"](https://techcabal.com/2025/11/10/equinix-to-build-22-million-data-centre-in-lagos/)
- [MainOne — Managed Private Cloud page (403 on direct fetch, reported via search summary)](https://mainone.net/mainone-cloud-solutions/managed-private-cloud/)
- [Businessday NG — OADC's "Open Access Fabric" cloud-access platform](https://businessday.ng/technology/article/oadcs-open-access-fabric-to-unlock-cloud-access-for-nigerias-200m-population/)
- [Businessday NG — UniCloud/OADC "sovereign cloud" partnership](https://businessday.ng/technology/article/africa-moves-to-end-data-colonialism-as-unicloud-oadc-seal-sovereign-cloud-deal/)
- [Leadership.ng — OADC's $240m/24MW Lagos expansion](https://leadership.ng/oadc-announces-240m-investment-in-24mw-lagos-data-centre/)
- [Liquid Intelligent Technologies — Africa Data Centres page](https://liquid.tech/data-centres/)
- [tech.africa — Africa Data Centres' new Lagos facility](https://tech.africa/africa-data-centres-lagos/)
- [Vanguard — "CBN Data Localisation: Galaxy Backbone moves to capture banks, fintechs" (Jul 2026)](https://www.vanguardngr.com/2026/07/cbn-data-localisation-galaxy-backbone-moves-to-capture-banks-fintechs/)
- [Galaxy Backbone — Services page](https://galaxybackbone.com.ng/services/)
- [AWS — official Local Zones list, confirming the Lagos zone (`af-south-1-los-1a`)](https://docs.aws.amazon.com/local-zones/latest/ug/available-local-zones.html)
- [Connecting Africa — "Lagos launches first AWS Local Zone in Africa"](https://www.connectingafrica.com/cloud-networking/lagos-launches-first-aws-local-zone-in-africa)
- [DEV Community — "Harnessing the Power of AWS Local Zones in Nigeria" (practical config guide, EC2-only coverage)](https://dev.to/fajmayor/harnessing-the-power-of-aws-local-zones-in-nigeria-a-server-configuration-guide-cec)
- [Google Cloud — Network edge locations documentation (confirms no Nigeria region)](https://docs.cloud.google.com/vpc/docs/edge-locations)

**CBN circular:**
- [Primary CBN circular PDF — located, 403 Forbidden on direct fetch](https://www.cbn.gov.ng/Out/2026/CCD/CIRCULAR%20ON%20INTRODUCTION%20OF%20MARKET%20STRUCTURE%20REQUIREMENTS,%20DATA%20LOCALISATION,%20ULTIMATE%20BENEFICIAL%20OWNERSHIP%20DISCLOSURE,%20AND%20SYSTEMIC%20OVERSIGHT%20MEASURES%20IN%20THE%20NIGERIA%20PAYMENTS%20SYSTEM.pdf)
- [Aluko & Oyebode — "CBN Issues Circular on Market Structure, Data Localisation, and Systemic Oversight"](https://www.aluko-oyebode.com/insights/cbn-issues-circular-on-market-structure-data-localisation-and-systemic-oversight-in-nigeria-payments/)
- [Enebeli and Partners Legal — circular summary with full addressee quote](https://goenebeli.com/cbn-circular-pss-dir-pub-cir-001-004-15-june-2026new-rules-on-data-localisation-beneficial-ownership-disclosure-market-concentration-in-the-nigerian-digital-payments-ecosystem/)
- [Kiet Business Attorneys — "The CBN Payments System Circular" client-alert PDF (read in full)](https://www.kietlaw.com/s/THE-CBN-CIRCULAR.pdf)
- [Kiet Law — blog version of the same alert](https://www.kietlaw.com/blog/the-central-bank-of-nigeria-payments-system-circular-market-structure-data-localisationbeneficial-ownership-amp-systemic-oversight)
- [Mondaq — "CBN's Data Localisation Directive: Compliance Considerations for Payment System Participants"](https://www.mondaq.com/nigeria/financial-services/1812264/cbns-data-localisation-directive-compliance-considerations-for-payment-system-participants)
- [Olaniwun Ajayi LP — "Decoding the CBN's New Market Structure and Data Localisation Directives"](https://www.olaniwunajayi.net/blog/decoding-the-cbns-new-market-structure-and-data-localisation-directives/)
- [allAfrica — "CBN Orders Fintechs, Banks to Localise Payment Data, Caps Market Dominance"](https://allafrica.com/stories/202606160033.html)
- [Legal500 — "The Cloud Is Not Borderless" (previously cited in T-037, re-reviewed here)](https://www.legal500.com/intelligence/nigeria/strategy/the-cloud-is-not-borderless-the-growing-importance-of-data-localisation-in-nigeria)
- [TechCabal — "CBN's local data order puts Nigeria's data centres to test" (previously cited in T-037)](https://techcabal.com/2026/07/07/cbns-local-data-order-puts-nigerias-data-centres-to-test/)

**Internal (read for context, not repeated):**
- `research/oss-core-banking-readiness-gaps.md` (T-037, especially G4)
- `research/nigerian-regulatory-requirements.md` (T-0xx, especially the "Needs compliance confirmation"
  section and its documented CBN-PDF access failures)
- `ARCHITECTURE.md` Section 9 (the "CBN data-localisation deadline" open question this brief updates) and
  P-5 (database-per-service, no shared DB — relevant to the Path B legal-coherence question in Section 4)
- `LedgerApi/docs/DISASTER-RECOVERY.md` (read directly — confirms today's exact Azure SKU, settings, and
  PITR behaviour used as the comparison baseline throughout this brief)
