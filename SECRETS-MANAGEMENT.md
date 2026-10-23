# Secrets management

T-042 (G7, from `research/oss-core-banking-readiness-gaps.md`): where GARBATI's production secrets
actually live today, and the deliberate choice to keep it that way for now rather than adopt a
centralized secrets-management service — stated explicitly, the same way `OBSERVABILITY.md` states the
"bring your own observability" posture, rather than left as a silent gap for someone to wonder about.

## What exists today, confirmed

- **API keys are never stored in plaintext, anywhere.** Every service stores only a SHA-256 hash of each
  client's key (`ARCHITECTURE.md` Section 5); the raw key exists only at issuance and in whoever received
  it. `LedgerApi`'s `scripts/demo-key.sh` issues/rotates/revokes per-visitor demo keys without ever
  printing the key to a terminal — it goes to the local Keychain, shared privately from there.
- **Production secrets (connection strings, API key hashes, signing keys) live as Azure App Service
  settings**, configured by hand (`az webapp config appsettings set` / `connection-string set`) by whoever
  has `az login` access. Confirmed directly: of GARBATI's 5 services, only 3 are actually deployed to Azure
  today (`ledgerapi`, `garbati-identityapi`, `garbati-orchestratorapi` — `Garbati.StaffIdentityApi` and
  `Garbati.OpsApi` don't have a working `deploy` CI job yet), and none of the three has a managed identity
  configured (`az webapp list` confirms `identity: null` on all three).
- **CI's own path to Azure already avoids a long-lived static secret.** Each service's `deploy` job
  authenticates to Azure via OIDC federated credentials (`client-id`/`tenant-id`/`subscription-id` GitHub
  secrets, no client secret or publish profile) — confirmed directly in `ci.yml`. This is already the
  "prefer short-lived/dynamic credentials" principle OWASP's guidance recommends, applied to the one leg
  of this system that most needed it.
- **Secrets are not duplicated into GitHub.** `LedgerApi/README.md`'s own line is accurate and confirmed:
  "the deploy job reads the connection string from [the App Service]; it is not stored in GitHub."

## What's missing

- **No single, centralized store.** Secrets are scattered across each service's own App Service settings
  panel individually — today 3 separate panels, growing to 5 once `Garbati.StaffIdentityApi`/`Garbati.OpsApi`
  deploy. There's no one place to see or manage everything at once.
- **No rotation schedule or mechanism beyond "someone runs a command."** `demo-key.sh` can rotate a demo
  API key on request; there's no equivalent for the production signing key, the database connection
  string's password, or any other long-lived credential — rotating any of those today means a manual
  `az` command run by whoever remembers to.
- **No confirmed audit trail of who changed a secret and when**, beyond whatever GitHub Actions' own run
  history shows for CI-driven changes. Checked directly (`az monitor activity-log list` against
  `ledger-rg`) and found no events for this resource group in the period checked — so this isn't something
  to claim as "already covered by Azure's platform logging" without direct evidence it actually captures
  these specific changes; treat it as unconfirmed, not assumed to be there.

## The real alternative considered, and why it's deferred

The natural "boring, no-new-infrastructure" answer, since GARBATI is already committed to Azure App
Service, is **Azure Key Vault with managed identity** — specifically via [Key Vault
references](https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references), an App
Service feature that resolves a setting written as `@Microsoft.KeyVault(SecretUri=...)` into the real
secret value at runtime, with **no SDK dependency and no code change in any service**. This lowers G7's
original effort estimate ("a real design task") — the actual integration is infra-only (create a vault,
grant each App Service's managed identity read access, point existing settings at vault references
instead of raw values), not an application-code change.

**Decided (owner, D-25, 23 Oct 2026): defer.** Not adopted now, deliberately — the same
stage-appropriate reasoning `ARCHITECTURE.md` P-6 already applies elsewhere ("solo developer and budget"):
a centralized vault, rotation policy and audit trail matter most once there's a team bigger than one
person, or real customer secrets at stake, neither of which is true of GARBATI today. The current
az-CLI-by-hand approach is a **named, deliberate choice**, not an oversight someone should "discover" and
flag again later as if it were new.

## When to revisit

- Before onboarding any real customer whose data or money is actually at risk from a leaked secret.
- Once more than one person regularly needs access to production secrets (today, exactly one person does).
- Once `Garbati.StaffIdentityApi`/`Garbati.OpsApi` actually deploy to Azure, since that's the point secrets
  management starts being a 5-service problem instead of a 3-service one.
- If G3 (BVN/NIN field-level encryption) is ever built — that decision will need its own managed-key
  infrastructure (likely Azure Key Vault too), which would make this decision worth revisiting at the same
  time rather than building two separate pieces of key-management infrastructure independently.
