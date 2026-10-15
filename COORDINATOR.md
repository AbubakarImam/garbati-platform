# Running the coordination chat

The coordination chat is where you decide, plan, and review. Agents build, check and research. The chat has no memory you can rely on, so **the files in this repo are the memory**. Start every coordination session by making the chat read them.

## Prompt to paste at the start of a session

> You are the coordinator for GARBATI. I am Abubakar, a C#/.NET learner; explain concepts in plain, non-technical words. Read `ARCHITECTURE.md`, `AGENTS.md` and `TASKS.md` in the garbati-platform repo. Your job: (1) keep `TASKS.md` and the decisions log accurate; (2) turn each task into a short task file in `tasks/` that an agent can follow; (3) review agent pull-request summaries and QA reports and tell me what I must decide; (4) assign decision numbers; (5) never decide for me on money rules, contracts or security: ask me. Before we end, quiz me on what was merged and update `LEARNING_LOG.md`. Say what you could not verify.

## Your loop
1. Pick the next task on the board whose dependencies are done and whose decisions you have made.
2. Ask the coordinator to write or refine the task file.
3. Start a builder agent on it in its own branch (git worktree). Start the QA agent when the PR opens.
4. Read the QA report. Answer the interview questions yourself before you merge.
5. Merge, then update `TASKS.md`, `ARCHITECTURE.md` (decisions status) and `LEARNING_LOG.md`.

Do not run more agents than you can review. Two builders plus QA is plenty.
