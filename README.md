# garbati-platform

The hub for GARBATI, an open-source core banking platform for Sharia-compliant microfinance. This repo holds no service code. It holds the map, the rules, the task board and the notes that keep every service and every agent in sync.

| File | What it is |
|---|---|
| `ARCHITECTURE.md` | Who owns which truth, who may call whom, the decisions log. **The source of truth.** |
| `TASKS.md` | The board: decisions you still need to make, and tasks for agents. |
| `AGENTS.md` | Rules every agent follows. |
| `COORDINATOR.md` | How to run the coordination chat (includes the prompt to paste). |
| `.claude/agents/` | Role files: builder, qa, researcher. |
| `tasks/` | One file per task, written for the agent that will do it. |
| `research/` | Researcher output, one file per question. |
| `qa/reports/` | QA reviews. |
| `contracts/` | OpenAPI file per service. Only the owner (you) approves changes. |
| `LEARNING_LOG.md` | After every merged feature: plain-language explanation and interview questions. |
| `templates/` | CI workflow and PR template to copy into each service repo. |
| `docker-compose.yml` | Runs every service, its own database and migrations together with one command. See below. |

Services live in their own repos: `LedgerApi`, `Garbati.IdentityApi`, `Garbati.OrchestratorApi`, `Garbati.StaffIdentityApi`, `Garbati.OpsApi`.

## Running the full stack locally

Clone every service repo above as a **sibling** of this one (same parent directory) - `docker-compose.yml` builds each from a relative path (`../LedgerApi`, etc.), so the layout matters:

```
some-parent-directory/
  garbati-platform/     <- you are here
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

This builds all five services' images, starts five PostgreSQL databases, runs each service's EF Core migrations (a one-shot `*-migrator` container per service that applies pending migrations and exits), then starts all five APIs wired to call each other over the compose network - no manual API-key setup or `localhost` port juggling needed. First build takes a while (five separate .NET builds); subsequent `docker compose up` runs reuse the cached images.

Once it's up:

| Service | URL |
|---|---|
| OrchestratorApi | `http://localhost:5001` |
| Garbati.IdentityApi | `http://localhost:5100` |
| LedgerApi | `http://localhost:5200` |
| Garbati.StaffIdentityApi | `http://localhost:5300` |
| Garbati.OpsApi | `http://localhost:5400` |

(OrchestratorApi is mapped to host port 5001, not 5000, because macOS's AirPlay Receiver already listens on 5000 on a stock Mac.)

Every dev API key and the signing key this stack relies on are the same ones already committed openly in each service's own `appsettings.Development.json` - see each service's own README for the plaintext dev keys (e.g. LedgerApi's `dev-payment-api` / `lk_dev_payment_...`, OrchestratorApi's `dev-demo-driver` / `dev-demo-driver-key`). None of this is a real secret; never reuse any of it outside your own machine.

The seeded `admin` Administrator account on StaffIdentityApi/OpsApi still needs the one-time bootstrap described in `Garbati.StaffIdentityApi`'s own README ("Bootstrapping the seeded Administrator") before its first login - this is unchanged by compose.

`docker compose down` stops and removes everything; add `-v` to also drop the database volumes (fresh databases on next `up`).
