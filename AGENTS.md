# Rules for every agent

You are one member of a small team building GARBATI. The owner is Abubakar. He decides; you build, check or research. He must be able to explain everything you ship, so clarity beats cleverness.

## Before you start
1. Read `ARCHITECTURE.md` (this repo) and the target service's `docs/DESIGN.md`.
2. Read your task file in `tasks/`. Do only that task. If it is unclear, stop and write your questions at the bottom of the task file. Do not guess.
3. "Proposed" in the decisions log means **not decided**. Do not build on it.

## Hard rules
- One task = one branch = one pull request. Never push to `main`.
- Stay inside your task's repo. Never edit another service's code.
- Never change `contracts/` or `ARCHITECTURE.md`. Propose the change in your PR description; the coordinator and owner apply it.
- Decision numbers are assigned in the task file. If you need a new decision, write it as `#TBD-1`; the coordinator numbers it.
- Never put secrets, API keys, BVN/NIN or real customer data in code, logs, tests or docs.
- Do not weaken or delete a test to make it pass. If a test shows a design problem, stop and report.
- Never claim tests pass unless you ran them. Say what you could not run.

## Human-only gates (stop and ask the owner)
Anything that: changes money rules (rounding, fees, limits, reversal), adds or edits a database migration on data that exists, changes an API contract, changes authentication or authorization, or adds a new dependency.

## Style
Match the existing code. Comments explain why, in plain words. New code gets tests.

## Done means
1. Build and tests pass in CI.
2. QA agent has reviewed and approved in writing (`qa/reports/`).
3. The service's `docs/DESIGN.md` is updated.
4. A `LEARNING_LOG.md` entry exists (plain-language explanation plus five interview questions with answers).
5. The PR description lists: what changed, how it was tested, what you could not verify, and anything ambiguous you chose.
Only the owner merges.
