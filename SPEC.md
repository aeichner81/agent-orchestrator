# Project: Agent Relay (a real multi-agent orchestrator)

A working command-line orchestrator that takes a `SPEC.md` (like the other files in this pack) and builds it using a team of specialised AI agents: an analyst, a planner, builders, a tester, a reviewer and an integrator. A live dashboard shows the agents working in parallel lanes, handing work to each other and sending it back when it isn't good enough. It is a real tool you will use to build the rest of this pack, and the dashboard and recorded runs are the showcase.

## Kick-off prompt (paste into Claude Code)

```
Read SPEC.md in full. It is the source of truth for this project.

Before writing any code, switch to plan mode and:
1. Check the current Claude Agent SDK (TypeScript) documentation and the
   Claude Code headless mode documentation. Summarise in
   docs/RESEARCH.md the APIs we will rely on (running a query, streaming
   messages, tool allow-lists, permission modes, hooks, cost/usage
   reporting, session resume), with links. Do not rely on memory for
   these APIs.
2. Ask me up to 5 clarifying questions about anything ambiguous in the spec,
   including the "Decisions to make" section.
3. Propose an implementation plan as numbered phases, following the phase
   outline in the spec. Each phase must end in something I can run.
4. Write the plan to docs/PLAN.md and wait for my approval.

Rules for the whole project:
- Do not add features that are not in SPEC.md. Suggest additions and wait
  for my agreement.
- All core logic must be testable with the FakeRunner (no API calls in
  tests, ever).
- After each phase: run tests, lint and build, then stop and summarise what
  changed so I can review and commit.
- Keep a running log of key decisions in docs/DECISIONS.md.
```

---

## SPEC

### Problem

A single AI coding agent working through a long spec loses focus, marks its own homework and has no second opinion. Real engineering teams split the work: someone plans, people build in parallel, someone else tests and someone reviews against the requirements. Agent Relay applies that team structure to AI agents, with humans approving at the points that matter, and makes the whole process visible.

### Who uses it

- **The operator (you):** runs Relay against a spec, answers questions, approves the plan, watches the dashboard and makes the final merge decision.
- **Viewers:** people watching a recorded run in the replay dashboard, for example from a LinkedIn link.

### The agent team

Each agent is defined in its own Markdown file with frontmatter (see "Agent definitions"). Defaults:

| Agent | Job | Can | Cannot |
|---|---|---|---|
| **Analyst** | Reads the spec, lists ambiguities and missing acceptance criteria as questions for the operator | Read files | Edit anything |
| **Planner** | Turns the spec and answers into a task graph (see "Task graph") and a `PLAN.md` | Read files, write `.relay/plan/` | Edit source code |
| **Builder** (one per task in progress) | Implements one task in its own git worktree, tests first | Read, edit, run allowed commands in its worktree | Edit `SPEC.md`, touch other worktrees, push |
| **Tester** | Independently checks the task's acceptance criteria: writes or extends tests, runs the full suite, reports pass/fail with evidence | Edit test files only, run tests | Edit non-test source files |
| **Reviewer** | Reviews the diff against the spec, the task's criteria and the project rules; returns Approve or Request changes with specific, numbered points | Read files and diffs | Edit anything |
| **Integrator** | Merges approved task branches into the integration branch, runs the full build and tests, and resolves simple conflicts or sends the task back | Git merge in the main repo, run commands | Push to a remote |

The orchestrator itself is plain TypeScript, not an agent. It decides who works next. This keeps the control flow predictable and testable.

### The pipeline

1. **Intake:** the operator runs `relay run --spec ./SPEC.md --repo ./target-folder`.
2. **Analyse:** the Analyst produces up to N questions. The CLI asks them one by one, and the answers are saved to `.relay/answers.md`.
3. **Plan:** the Planner produces the task graph and `PLAN.md`.
4. **Gate 1, plan approval:** the CLI shows the plan. The operator approves, edits the file and re-runs, or rejects.
5. **Execute:** the orchestrator runs every task whose dependencies are done, up to `maxParallel` at once (default 2). Each task goes through this loop:
   - Builder implements the task in worktree `relay/task-<id>`.
   - Tester verifies it. If it fails, the failure report goes back to the Builder.
   - Reviewer reviews it. If it requests changes, the numbered points go back to the Builder.
   - Each task gets at most `maxReviewRounds` (default 2) rejection loops. After that it is marked **Needs human** and the operator is prompted.
   - Integrator merges the approved task into the `relay/integration` branch and runs the full suite. If that fails, the task goes back to the Builder with the failure.
6. **Gate 2, final review:** the operator sees a summary (tasks, rounds, cost, test results, diff stats) and decides whether to merge `relay/integration` into their main branch. Relay never pushes.
7. **Report:** Relay writes `.relay/runs/<run-id>/REPORT.md` with the timeline, every handoff, every rejection and its reason, cost per agent and lessons ("tasks that needed the most rounds").

### Task graph

The Planner must output JSON that passes validation (use zod or similar) before the run continues:

```json
{
  "tasks": [
    {
      "id": "T3",
      "title": "Throughput CSV import",
      "description": "…",
      "acceptanceCriteria": ["AC-4", "AC-5"],
      "dependsOn": ["T1"],
      "likelyFiles": ["src/import/*"],
      "size": "S | M | L"
    }
  ]
}
```

Validation rules: unique IDs, no dependency cycles, every acceptance criterion in the spec is covered by at least one task (report any that aren't), and no task is sized L (the Planner must split those). If validation fails, the errors go back to the Planner, with up to 2 retries.

### Agent definitions (the reusable part)

- The default agents live in `agents/*.md` in the Relay repo. Each file has frontmatter (`name`, `role`, `model`, `allowedTools`, `maxTurns`, `maxBudgetUsd`) and the body is the system prompt.
- A target project can override or add agents in its own `.relay/agents/` folder, for example a stricter Reviewer for a particular codebase.
- The pipeline is configured in `relay.config.json` (in Relay, overridable in the target): stage order, gates on or off, `maxParallel`, `maxReviewRounds`, budget caps, and the commands for test, lint and build.
- Agent prompts must tell each agent to read the target project's `CLAUDE.md` if it has one.

### Runners

All agent calls go through a `Runner` interface so the engine doesn't care how an agent runs:

- **AgentSdkRunner:** uses the Claude Agent SDK. Needs an API key and is billed per token.
- **ClaudeCodeRunner:** calls Claude Code in headless mode as a child process, with streaming JSON output. It can use the operator's existing Claude Code login.
- **FakeRunner:** replays scripted responses from fixture files. Used in every test and in "demo mode".

Every runner emits the same normalised events (see "Events").

### Events and state

- Every action appends one JSON line to `.relay/runs/<run-id>/events.jsonl`: run started, agent started, tool used, message, handoff, verdict, gate, merge, cost update, error, run finished. Each event has a timestamp, agent, task ID and payload.
- The run state (`run.json`) can be rebuilt entirely from the events. `relay resume <run-id>` continues a run that was interrupted.
- Full agent transcripts are saved per agent per task.

### Guardrails (non-negotiable)

- Tool permissions are enforced per agent, using the SDK's tool allow-lists and hooks or the CLI's equivalent flags, not just by asking nicely in the prompt.
- A hook blocks dangerous commands for every agent: `rm -rf` outside the worktree, `git push`, `curl | sh`, editing anything outside the target repo, and reading `.env` files.
- Budget caps per agent call and per run, in dollars and turns. When a cap is reached the run pauses at a safe point and asks the operator.
- Relay refuses to run on a target repo with uncommitted changes.
- Relay works only on `relay/*` branches and never rewrites the operator's history.

### Dashboard

A local web app started with `relay ui`, which reads events live through server-sent events from a small local server.

- **Swimlanes:** one lane per agent instance, with blocks showing what each is doing. Handoffs are drawn as arrows between lanes, and rejections are drawn as red return arrows labelled with the reason.
- **Task board:** columns for Queued, Building, Testing, In review, Integrating, Done and Needs human. Each card shows the round count.
- **Detail pane:** click any block to see the prompt summary, tool calls, diff and verdict.
- **Meters:** cost so far against budget, tokens per agent, elapsed time and tasks done out of the total.
- **Replay mode:** load any `events.jsonl` and scrub through it with play, pause and speed controls.
- **Static replay build:** `npm run build:replay -- <run-id>` produces a static site with one recorded run built in. It can be deployed to GitHub Pages so anyone can watch the run without a key.

### Decisions to make (ask me during planning)

- Which runner is the default (cost compared with simplicity).
- Whether Builders get one worktree per task (cleaner) or one per Builder slot (faster setup).
- Which model each agent uses by default (for example, a cheaper model for the Analyst and Tester).

### Tech

- Node 20+ and TypeScript, as an npm workspaces monorepo: `packages/core` (engine, graph, events, runners), `packages/cli`, and `packages/dashboard` (Vite + React + TypeScript, plain CSS).
- Vitest for tests. The core engine is tested with the FakeRunner, covering scheduling, dependency order, parallel limits, rejection loops, budget pauses, resume and graph validation.
- `fixtures/sample-spec/` holds a tiny spec (a tip calculator) used for smoke runs.
- GitHub Actions runs tests and deploys the static replay site to GitHub Pages.

### Phase outline

1. Engine, task graph validation and event log with the FakeRunner. Running a sample spec in demo mode works end to end with no API calls.
2. A real runner and a single sequential task on the sample spec.
3. Tester and Reviewer loop, both gates, budget caps and guardrail hooks.
4. Worktrees and parallel tasks, the Integrator, and resume.
5. Live dashboard.
6. Replay mode, static replay build and GitHub Pages deployment.
7. Dogfooding: use Relay to build `07-delivery-forecaster.md` and record the run.

### Acceptance criteria

- Given the sample spec and the FakeRunner, when I run `relay run --demo`, the whole pipeline completes and `events.jsonl` contains the full sequence, including at least one scripted rejection that is fixed in round 2.
- Given a task graph with a dependency cycle or an uncovered acceptance criterion, the Planner's output is rejected with a clear error and retried.
- Given `maxParallel: 2` and four independent tasks, no more than two Builders are ever active at once (verified from events).
- Given a Reviewer that rejects 3 times with `maxReviewRounds: 2`, the task goes to Needs human and the run continues with other tasks.
- Given an agent that tries `git push` or edits a file outside its worktree, the action is blocked and logged, and the run continues.
- Given a run killed halfway, when I run `relay resume`, it continues from the last completed step without redoing finished tasks.
- Given a real run on the sample spec, the result builds, passes its tests and has a `REPORT.md` with cost per agent.
- The replay site plays a recorded run on GitHub Pages with no backend.
- `npm test` makes zero network calls.

### Out of scope

Pushing to remotes or opening pull requests (stretch goal), non-Claude models, a hosted multi-user service, and agents choosing their own workflow (the orchestrator decides the flow).

### README must include

A one-paragraph pitch, a GIF of the swimlane dashboard showing a rejection loop, a link to the live replay, an architecture diagram of the pipeline and gates, the agent roster table, the guardrails, how to define your own agents, "Built X with Relay" with numbers (tasks, rounds, cost, time), honest lessons, and a "How this was built with Claude Code" section linking to docs/PLAN.md and docs/DECISIONS.md.
