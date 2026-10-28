---
layout: default
title: Replicate It
permalink: /replicate/
---

# Run the whole platform yourself

GARBATI runs locally with one command. Clone every service repo as a **sibling** of
`garbati-platform` (same parent directory):

```
some-parent-directory/
  garbati-platform/     <- this repo
  LedgerApi/
  Garbati.IdentityApi/
  Garbati.OrchestratorApi/
  Garbati.StaffIdentityApi/
  Garbati.OpsApi/
```

Then, from `garbati-platform/`:

```bash
docker compose up --build
```

This builds all five services' images, starts five PostgreSQL databases, runs each service's EF
Core migrations automatically, then starts all five APIs wired to call each other over the compose
network — no manual API-key setup, no `localhost` port juggling. First build takes a while (five
separate .NET builds); subsequent runs reuse the cached images.

<div class="ledger-rule"><span>What's below</span></div>

The rest of this page mirrors `DEPLOYMENT-RUNBOOK.md` from the repository root — the same, exact
runbook used to take real services from "built and tested" to "actually running" on Azure, not a
separate, simplified version written for this site. It's kept in sync automatically by
[`.github/workflows/docs-sync.yml`](https://github.com/AbubakarImam/garbati-platform/blob/main/.github/workflows/docs-sync.yml)
on every push that changes the real file, not hand-copied — see that workflow and
`scripts/sync_docs.py` for why a GitHub Action does this instead of a plain Jekyll tag (Jekyll's
own `include`/`include_relative` tags refuse any path containing `..`, so neither can reach a file
one directory above `docs/` directly). Prefer reading the real file itself?
[It's directly on GitHub](https://github.com/AbubakarImam/garbati-platform/blob/main/DEPLOYMENT-RUNBOOK.md).

{% include deployment-runbook.md %}
