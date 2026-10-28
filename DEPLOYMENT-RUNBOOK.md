# Deploying a GARBATI service to Azure

D-27 (G11, from `research/oss-core-banking-readiness-gaps.md`): the exact, repeatable sequence used to
take `Garbati.StaffIdentityApi`, `Garbati.OpsApi` and `Garbati.ProductApi` from "built and tested" to
"actually running" (T-048, 27 Oct 2026) — written down so the next service doesn't need this
reconstructed from memory or chat history. **Deliberately not Infrastructure-as-Code** (Terraform/Bicep/
Pulumi) — the owner's decision: a plain runbook now, real IaC once the remaining services
(`ScheduleApi`, financing services, `Channel`/`Gateway`) are built. See "When to actually move to IaC"
at the end.

## What's shared across every service (set up once, not per-service)

All six services deployed so far share:
- **One resource group**: `ledger-rg`.
- **One PostgreSQL Flexible Server**: `ledgerapi-db-san` — each service gets its own *database* on this
  server, never its own server. Confirmed deliberately cost-saving, not an oversight.
- **One App Service Plan**: `ASP-ledgerrg-8f31` (Basic B1) — each service gets its own *site* on this
  plan, never its own plan.

Do not provision a new server or plan for a new service without a real reason (a genuine resource
contention problem, not just "it feels cleaner"). Adding a service to this pattern is just one more
database and one more site on infrastructure that already exists.

## Per-service steps, in order

Assume `$APP_NAME` (e.g. `garbati-scheduleapi`), `$DB_NAME` (lowercase, no dashes, e.g. `scheduleapi`),
`$CONN_STRING_NAME` (the exact name the service's own `ConnectionStrings` section expects, e.g.
`ScheduleDb`), and `$REPO` (e.g. `Garbati.ScheduleApi`) below.

### 1. Create the database

```bash
az postgres flexible-server db create --resource-group ledger-rg --server-name ledgerapi-db-san --name "$DB_NAME"
```

### 2. Create the App Service

```bash
az webapp create --resource-group ledger-rg --plan ASP-ledgerrg-8f31 --name "$APP_NAME" --runtime "DOTNETCORE:8.0"
```

Verify the runtime actually took (`az webapp config show --name "$APP_NAME" --resource-group ledger-rg
--query linuxFxVersion -o tsv` should print `DOTNETCORE|8.0`) — it has silently shown blank before.

### 3. Set the connection string

```bash
az webapp config connection-string set --name "$APP_NAME" --resource-group ledger-rg --connection-string-type Custom \
  --settings "$CONN_STRING_NAME=Host=ledgerapi-db-san.postgres.database.azure.com;Port=5432;Database=$DB_NAME;Username=ledgerapidb;Password=<the shared admin password>;SSL Mode=VerifyFull"
```

### 4. Create the Azure AD app registration + service principal for GitHub OIDC

```bash
az ad app create --display-name "$APP_NAME-deploy"              # note the appId returned
az ad sp create --id "<appId>"
```

### 5. Add the federated credential — **read this before guessing the subject format**

**The subject is NOT `repo:OWNER/REPO:ref:refs/heads/main`.** GitHub's real OIDC subject claim includes
immutable numeric account and repo IDs, appended with `@`. Get them first:

```bash
gh api user --jq '.id'                          # account id, same for every repo under this account
gh api repos/AbubakarImam/$REPO --jq '.id'       # this specific repo's id
```

Then:

```json
{
  "name": "github-main-deploy",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:AbubakarImam@<account id>/<REPO>@<repo id>:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}
```

```bash
az ad app federated-credential create --id "<appId>" --parameters federated-credential.json
```

Getting this wrong surfaces as `AADSTS700213` on the first deploy attempt, with the *actual* expected
subject printed in the error — if you do guess wrong, that error message is the fastest way to the real
value, faster than re-deriving it from `gh api` first.

### 6. Grant the role assignment — **this step needs the owner, not the agent**

```bash
az role assignment create --assignee "<appId>" --role "Contributor" --scope=/subscriptions/<subscription id>/resourceGroups/ledger-rg
```

Granting management-plane Azure access to a new identity is blocked for the agent by this session's own
permission classifier, correctly. The owner runs this one directly (prefix with `!` in the Claude Code
prompt to execute in their own session).

### 7. Set the three GitHub secrets

```bash
gh secret set AZUREAPPSERVICE_CLIENTID_<SERVICE> --repo AbubakarImam/$REPO --body "<appId>"
gh secret set AZUREAPPSERVICE_TENANTID_<SERVICE> --repo AbubakarImam/$REPO --body "<tenant id>"
gh secret set AZUREAPPSERVICE_SUBSCRIPTIONID_<SERVICE> --repo AbubakarImam/$REPO --body "<subscription id>"
```

### 8. Author (or verify) the `deploy` job in `ci.yml`

Copy the exact shape already used by every sibling's `deploy` job (`needs: [test, docker]`, publish →
zip → migration bundle → `azure/login` → read the connection string from the App Service itself (never
duplicate it into GitHub) → open/close a scoped Postgres firewall rule → apply migrations → zip-deploy →
smoke test both `/health/ready` and an unauthenticated request to a real protected route). Don't
improvise a different shape — read a sibling's actual `.github/workflows/ci.yml` directly and mirror it.

### 9. Provision real (non-dev) API keys for every genuine cross-service call

For each service this one actually calls, or that actually calls this one in production:

- Generate: `KEY="<prefix>_$(openssl rand -base64 32 | tr '+/' '-_' | tr -d '=')"`.
- Hash: `printf '%s' "$KEY" | openssl dgst -sha256 -binary | base64`.
- Add the hash as a **new, additive** `ApiKeys__Clients__N__*` entry on the *receiving* service's App
  Service settings — never overwrite an existing index, never touch an existing client another service
  already depends on. Take the scopes from that service's own `DESIGN.md` scope table, not from a
  same-ClientId entry that happens to already exist elsewhere — one already turned out to have the wrong
  scopes for this exact purpose.
- Set the plaintext key + the real base URL as `DownstreamServices__<Service>__*` on the *calling*
  service's own settings.
- **Never leave the plaintext sitting in a scratch file longer than the single session that needed it.**

### 10. Trigger the deploy and verify independently

```bash
gh workflow run ci.yml --repo AbubakarImam/$REPO --ref main   # or just `git push`, either triggers it
gh run watch <run-id> --repo AbubakarImam/$REPO --exit-status
```

Then verify yourself, separately from the deploy job's own smoke test:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "https://$APP_NAME.azurewebsites.net/health/ready"   # expect 200
curl -s -o /dev/null -w "%{http_code}\n" "https://$APP_NAME.azurewebsites.net/<a real protected route>"  # expect 401
```

## Known gotchas, confirmed directly against real behaviour

- **`az webapp restart` does not reliably reload new App Service settings into the running process.**
  The app reports healthy and keeps serving the *old* configuration. Use a full `az webapp stop` then
  `az webapp start` after any settings change you need to actually take effect before the next check.
- **A service's default hostname can change on its own.** Azure has a platform-wide policy of migrating
  older App Services without a custom domain to a unique, suffixed default hostname
  (`<app>-<suffix>.<region>-01.azurewebsites.net`) instead of the plain `<app>.azurewebsites.net`, as an
  anti-subdomain-takeover measure — `LedgerApi` was migrated this way at some point without any action on
  this platform's part. **Before trusting any previously-documented service URL, confirm it still
  resolves and matches `az webapp show --query defaultHostName` for that app right now.** This exact
  class of drift — a setting that was correct when written, silently wrong later — is the strongest real
  argument *for* eventually adopting real IaC with drift detection, not a hypothetical one.
- **Don't assume an existing, similarly-named credential is reusable.** A client with a plausible name
  may already exist on a target service with the wrong scopes for your actual need, and its plaintext is
  never recoverable (hash-only storage, by design) even if the scopes did match. Check the actual scopes
  against the calling service's own `DESIGN.md`, and generate fresh if there's any doubt.
- **A migration that adds a new unique/filtered constraint to an already-populated table can fail at
  deploy time if real data already violates it — and that is the *correct*, safe outcome, not a bug to
  work around.** Confirmed directly (T-049, D-16, 28 Oct 2026): `IdentityApi`'s deploy failed its `Apply
  migrations` step with a genuine Postgres `23505` unique-violation against one pre-existing duplicate
  row. The deploy aborted cleanly, the running app was never touched, and nothing was corrupted. The
  agent could not query or fix the production data itself — the session's own permission classifier
  blocks any direct read or write against a production database, the same class of restriction as the
  RBAC role-assignment step above. Pattern that worked: the owner opens a temporary, IP-scoped firewall
  rule (`az postgres flexible-server firewall-rule create` — note a residential IP can change between
  steps; re-run `curl -s https://api.ipify.org` and `az ... firewall-rule update` if a connection times
  out), the agent writes a **self-contained script** (not an inline one-off command) that the owner runs
  via `!` — one script to inspect and report the actual conflicting rows, a second, separate script to
  resolve them that **re-verifies the exact row's full contents against the values just confirmed before
  deleting anything**, aborting instead of deleting on any mismatch. The owner gives explicit go-ahead
  between the two scripts; the agent never deletes data from a judgment call alone. Firewall rule removed
  afterward. Only once the conflict is actually resolved does re-running the deploy make sense — retrying
  blindly just reproduces the same failure.

## When to actually move to Infrastructure-as-Code

Owner's decision, 27 Oct 2026 (D-27): **not now.** Revisit once the remaining never-started services
(`ScheduleApi`, financing services, `Channel`/`Gateway`) are actually built and this runbook has been
run a few more times — by then there will be more resources to manage, more repeatable pain to justify
the tooling investment, and a clearer picture of what a Terraform/Bicep module for this platform
actually needs to express, rather than guessing at its shape now from six resources.
