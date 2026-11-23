# Decisions log

| # | Date | Decision | Status | Notes |
|---|---|---|---|---|
| D1 | 2026-11-22 | Default runner is ClaudeCodeRunner (headless `claude -p`, Pro subscription). AgentSdkRunner is also built but opt-in only | Agreed | Operator wants to avoid per-token billing. Pro has no included API credits. See RESEARCH §1. |
| D2 | 2026-11-22 | One git worktree per task (`relay/task-<id>`) | Agreed | Clean per-task diffs and simpler resume, at the cost of per-task setup time |
| D3 | 2026-11-22 | Models: Planner opus. Analyst and Tester haiku. Builder, Reviewer and Integrator sonnet | Agreed | Set in agent frontmatter, can be overridden per target |
| D4 | 2026-11-22 | Runs natively on Windows (PowerShell). CI on Ubuntu | Agreed | Node-based guard hook. Path normalisation tested |
| D5 | 2026-11-22 | Caps: $3 per call, $25 per run (estimated), 40 turns per call. Analyst asks at most 5 questions | Agreed | On headless these are equivalent API cost, not real spend |
| D6 | 2026-11-21 | Guard is a pure function shared by both runners (in-process for the SDK, Node hook script for the CLI) | Proposed | One implementation, one set of tests |
| D7 | 2026-11-22 | On a `rate_limit` from the Pro plan, pause at a safe point and resume later | Proposed | `maxParallel: 1` is the fallback if limits bite |
| D8 | 2026-11-22 | `specs/001-agent-relay.md` is the formal spec and the source of truth for US, AC and NFR IDs. `SPEC.md` is kept as the original brief | Agreed | Matches the house spec format (portfolio guide 04). Conflicts between the two files go to the owner. |
| D9 | 2026-11-23 | Planner still invalid after 2 retries: stop before Gate 1 and show the validation errors (spec Q1) | Agreed | Nothing is built |
| D10 | 2026-11-23 | Needs human options: retry with an operator note (resets rounds once), skip the task, or stop the run (Q2) | Agreed | |
| D11 | 2026-11-23 | Tasks depending on a Needs-human task stay blocked and are listed at Gate 2 (Q3) | Agreed | |
| D12 | 2026-11-23 | `maxParallel` limits tasks in progress at once, across Builders, Testers and Reviewers (Q4) | Agreed | Satisfies AC-3 |
| D13 | 2026-11-23 | Integrator may edit only conflicting sections, and only if the full suite then passes; otherwise the task goes back to its Builder (Q5) | Agreed | |
| D14 | 2026-11-23 | Analyst finds no ambiguities: no questions, empty `answers.md`, go straight to planning (Q6) | Agreed | |
