# Agent Relay: implementation plan

Status: **Approved by the operator.** Version 2, updated 2026-11-22 with the operator's answers. Each phase is still planned in plan mode before any of its code is written.
Source of truth: `specs/001-agent-relay.md` (user stories US-1 to US-14, acceptance criteria AC-1 to AC-41), taken from the original brief in `SPEC.md`. API details: `docs/RESEARCH.md`. Decisions: `docs/DECISIONS.md`.
Every phase and task names the AC IDs it covers. Questions Q1–Q6 in spec section 8 were resolved on 2026-11-23 (D9–D14). Q1–Q4 and Q6 apply from Phase 1, and Q5 from Phase 4.

## Agreed defaults (operator answers, 2026-11-22)

| Decision | Proposal | Why |
|---|---|---|
| Default runner | **ClaudeCodeRunner** (`claude -p`, headless, Pro subscription login) | Operator choice. No per-token bill. Has nearly the same features as the SDK for what the spec needs. |
| AgentSdkRunner | Built as well, behind the same interface (Phase 3), **opt-in only** | Required by SPEC "Runners". On Pro there are no included API credits, so this runner is pure pay-as-you-go and is never selected unless `runner: "sdk"` is set explicitly. |
| Worktrees | **One per task** (`relay/task-<id>`) | Operator choice. Cleaner diffs and reviews. Makes resume simpler. |
| Models | Planner: **opus**. Analyst and Tester: **haiku**. Builder, Reviewer and Integrator: **sonnet** | Operator choice. Set in `agents/*.md` frontmatter, so it can be overridden per target. |
| `maxParallel` / `maxReviewRounds` | 2 / 2 | Spec defaults |
| Analyst question limit N | 5 | The spec leaves N open |
| Budget caps (estimated USD) | $3 per agent call, $25 per run, 40 turns per call | Agreed. On the headless runner this is an equivalent API cost, not money spent. |
| Platform | **Native Windows (PowerShell)** for running it. CI on Ubuntu. | Operator choice. Guard hooks are written in Node, paths are normalised, and tests cover Windows-style paths and paths with spaces. |
| Usage limits | Pro plan: pause and resume when limits are hit | 2 parallel agents can reach the Pro 5-hour limit. Relay detects `rate_limit`, pauses, shows when to resume, and `relay resume` continues. |

## Architecture

```
packages/
  core/        engine (scheduler, pipeline state machine), graph (zod + validators),
               events (append-only JSONL + reducer -> run.json), runners/, guard/, config/, git/
  cli/         `relay run | resume | ui`, prompts for questions and gates
  dashboard/   Vite + React + TS + plain CSS; live (SSE) + replay
agents/        analyst.md planner.md builder.md tester.md reviewer.md integrator.md
relay.config.json
fixtures/sample-spec/      tip calculator spec
fixtures/fake-runs/        scripted FakeRunner responses (incl. a round-2 rejection)
```

Key ideas:
- **The orchestrator is plain TypeScript**: a pure state machine `(state, event) -> commands`. That makes scheduling, rejection loops, gates, budget pauses and resume testable with no I/O.
- **Event sourcing**: every state change is an event appended to `events.jsonl` first. `run.json` = `reduce(events)`. Resume = replay, then continue.
- **Runner interface**: `run(agentCall, signal) => AsyncIterable<RelayEvent>`. Real runners map SDK or CLI messages to Relay's normalised events, so the engine can't tell them apart.
- **One guard, two transports**: `guard/check.ts` is a pure function `(toolName, input, ctx) => allow | deny(reason)`. The SDK runner calls it in-process. The CLI runner calls it through a small Node hook script passed with `--settings`.

## Phases

Each phase ends with tests, lint and build passing, a summary, and a stop for you to review and commit.

### Phase 1: Engine, task graph, event log, FakeRunner (demo mode)
- npm-workspaces monorepo, TS strict, ESLint, Vitest, GitHub Actions running test, lint and build.
- `core/graph`: zod schema plus validators (unique IDs, cycle detection, AC coverage, no L-sized tasks) and a Planner retry loop (up to 2).
- `core/events`: typed events, JSONL writer, reducer, `run.json`.
- `core/engine`: pipeline (analyse, plan, gate 1, execute loop, gate 2, report), scheduler respecting `dependsOn` and `maxParallel`, `maxReviewRounds` leading to Needs human, budget accounting and pause.
- `FakeRunner` plus fixtures for the tip-calculator spec, including a scripted Reviewer rejection fixed in round 2.
- `relay run --demo` (gates answered automatically in demo mode) and a basic `REPORT.md`.
- **You can run:** `npm test` and `npx relay run --demo` → `.relay/runs/<id>/events.jsonl` and `REPORT.md`.
- ACs: demo pipeline; cycle or uncovered AC rejected and retried; maxParallel=2 with 4 tasks; 3 rejections leading to Needs human; `npm test` makes zero network calls (enforced by a test-setup guard that fails on any socket or child process).

### Phase 2: Real runner, one sequential task
- `ClaudeCodeRunner`: spawns `claude -p` with `--output-format stream-json --verbose`, `--tools`, `--allowedTools`, `--disallowedTools`, `--permission-mode dontAsk`, `--max-turns`, `--max-budget-usd`, `--model`, `--system-prompt-file`, `--setting-sources project`, `--strict-mcp-config`, and `--json-schema` for the Planner. Maps stream lines to Relay events. Detects `api_retry` with `rate_limit` and pauses.
- Agent definitions loaded from `agents/*.md` (frontmatter plus body), with overrides from the target's `.relay/agents/`. Prompts tell agents to read the target's `CLAUDE.md`.
- Real Analyst and Planner, plus one Builder working directly on a `relay/integration` branch (no worktrees yet) on the sample spec, with `maxParallel` forced to 1.
- Parser tests use recorded stream-json fixtures, so there are still no network calls.
- **You can run:** `relay run --spec fixtures/sample-spec/SPEC.md --repo <scratch-repo>` and watch it answer questions, plan and build task 1.

### Phase 3: Tester and Reviewer loop, gates, budgets, guardrails, AgentSdkRunner
- Tester (test files only) and Reviewer (read-only) agents. Failure reports and numbered review points go back to the Builder.
- Gate 1 (approve, or edit-and-re-run, or reject) and Gate 2 (summary of tasks, rounds, cost, tests and diff stats; the operator merges, Relay never pushes).
- Budget caps per call and per run, which pause at a safe point and ask the operator.
- Guard: blocks `rm -rf` outside the worktree, `git push`, `curl|sh` / `wget|sh`, Write or Edit outside the target repo, and reading `.env*` (via Read and via Bash). Every block is logged as an event. Defence in depth: `disallowedTools` hard rules plus the PreToolUse hook.
- Refuses to run on a dirty target repo. Only touches `relay/*` branches.
- `AgentSdkRunner` behind the same interface, sharing the same guard function.
- **You can run:** a full sequential real run on the sample spec with both gates, and a scripted "agent tries `git push`" test that is blocked, logged, and the run continues.

### Phase 4: Worktrees, parallel tasks, Integrator, resume
- `git worktree add` per task, with Builders and Testers scoped to their worktree (`cwd`, guard root).
- Integrator merges approved branches into `relay/integration`, runs the full suite, and sends the task back on failure or a non-trivial conflict.
- `relay resume <run-id>` rebuilds from events, skips finished tasks, and resumes interrupted agent sessions where possible.
- Full transcripts saved per agent per task.
- **You can run:** a real run with 2 parallel Builders. Kill it halfway, run `relay resume`, and it finishes without redoing done tasks.

### Phase 5: Live dashboard
- `relay ui` starts a small local server: SSE tail of `events.jsonl` plus static dashboard.
- Swimlanes (agent instances, handoff arrows, red rejection arrows with the reason), task board (7 columns with round counts), detail pane (prompt summary, tool calls, diff, verdict), meters (cost against budget, tokens per agent, elapsed time, tasks done out of total).
- **You can run:** `relay ui` alongside a demo or real run and watch it live.

### Phase 6: Replay, static build, GitHub Pages
- Replay mode: load any `events.jsonl`, with play, pause, speed and a scrubber. It uses the same reducer as the engine, so there's no separate logic.
- `npm run build:replay -- <run-id>` builds a static site with the run embedded, and a GitHub Actions job deploys it to Pages.
- **You can run:** open the Pages URL and watch a recorded run with no backend.

### Phase 7: Dogfooding
- Point Relay at `07-delivery-forecaster.md`, record the run, publish the replay, and fill in the README sections from the spec (pitch, GIF, replay link, architecture diagram, roster, guardrails, custom agents, "Built X with Relay" numbers, lessons, how it was built).
- **You can run:** the forecaster repo built by Relay, plus the public replay.

## Acceptance-criteria traceability

| Phase | ACs delivered | How verified |
|---|---|---|
| 1. Engine, graph, events, FakeRunner | AC-1, AC-2, AC-3, AC-4, AC-6 (engine level), AC-9, AC-11 (fake), AC-12, AC-13, AC-14, AC-15, AC-17, AC-18, AC-27, AC-28 (basic), AC-30, AC-31 | Vitest with the FakeRunner. `relay run --demo`. AC-3 and AC-15 are checked by assertions over the event timeline. AC-9 by a test-setup trap on sockets and child processes. |
| 2. Real runner, one sequential task | AC-10, AC-11 (real), AC-32, AC-34, AC-33 (ClaudeCodeRunner parser) | Parser tests on recorded stream-json. A manual run on the sample spec. |
| 3. Test and review loop, gates, budgets, guardrails, SDK runner | AC-5, AC-20, AC-21, AC-22, AC-23, AC-24, AC-25, AC-26, AC-33 (SDK parser), AC-7 (sequential) | Guard unit tests (including Windows paths). Scripted FakeRunner attempts. A full manual run. |
| 4. Worktrees, parallel, Integrator, resume | AC-16, AC-19, AC-6 (real), AC-29, AC-3 (re-checked on a real run), AC-7 (parallel) | Integration tests against a temporary git repo. Manual kill-and-resume. |
| 5. Live dashboard | AC-35, AC-36, AC-37, AC-38, AC-39 | Component tests on recorded events. Manual check during a live run. |
| 6. Replay and Pages | AC-40, AC-41, AC-8 | Replay tests on recorded events. Deployed URL. |
| 7. Dogfooding | AC-28 (full), NFR-9 | A recorded forecaster run and the README checklist |

## Risks

- **Pro plan limits (likely, not just possible)**: 2 parallel agents plus review loops will probably hit the 5-hour limit on a full real run. Mitigation: detect `rate_limit`, pause cleanly and tell you, and resume. Haiku for the Analyst and Tester saves usage, and the Planner (opus) runs only once or twice per run. Fallback if it hurts: set `maxParallel: 1` for real runs (config only). Phase 7 (the forecaster) may span more than one limit window.
- **CLI drift**: the stream-json format changes between versions. Mitigation: record the `claude --version` in `run.started`, use tolerant parsing, and keep recorded fixtures for parser tests.
- **Billing policy changes**: Anthropic announced and then paused changes this year. Mitigation: the Runner interface means switching is a config change.
- **Windows specifics**: path separators in the guard, git worktree paths with spaces (your folder has spaces). Mitigation: tests that use Windows-style paths.
- **Your personal Claude Code config leaking into agents**: mitigated by the explicit flags (RESEARCH §10).

## Out of scope (per spec)

Pushing or PRs, non-Claude models, a hosted multi-user service, agents choosing their own workflow.
