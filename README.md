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

Services live in their own repos: `LedgerApi` (built), `Garbati.IdentityApi`, `Garbati.ProductApi`, `Garbati.OrchestratorApi`.
