# Spec 001: Agent Relay (multi-agent orchestrator)
Status: Approved (owner, 2026-11-23)
Revision 1 (2026-11-22): restructures `SPEC.md` (portfolio spec 05) into the house format. It adds numbered user stories, numbers the original acceptance criteria AC-1 to AC-9 (wording unchanged), and adds AC-10 onwards, each taken from a statement already in `SPEC.md` (the section is cited). It records the operator's answers to "Decisions to make". No new features. Section 8 lists the open questions that need an owner decision before they can be specified.

`SPEC.md` stays as the original brief. **This file is the source of truth for IDs (US-x, AC-x, NFR-x).** Tasks, tests and the Planner's coverage check refer to these IDs. If the two files disagree, raise it with the owner. Do not choose one silently.

## 1. Problem
A single AI coding agent working through a long spec loses focus, marks its own homework and gets no second opinion. Real engineering teams split the work: one person plans, several build in parallel, someone else tests, and someone reviews against the requirements. Agent Relay applies that team structure to AI agents. Humans approve at the points that matter, and the whole process is visible on a dashboard. Relay is a real tool used to build the rest of the portfolio pack. The dashboard and recorded runs are the showcase.

## 2. Users
- **Primary: the operator (the owner).** Runs Relay against a spec, answers questions, approves the plan, watches the dashboard and makes the final merge decision. Also customises agents and pipeline settings per target project.
- **Secondary: viewers.** People watching a recorded run in the replay dashboard, for example from a LinkedIn link. They have no API key and no local setup.

## 3. User stories
- US-1: As an operator, I want to start a run with one command against a spec and a target repo, so that a team of agents builds the spec for me.
- US-2: As an operator, I want the Analyst to ask me about ambiguities and missing acceptance criteria before planning, so that agents don't build on guesses.
- US-3: As an operator, I want to approve, edit or reject a validated plan (task graph) before any code is written, so that I control what gets built (Gate 1).
- US-4: As an operator, I want independent tasks built in parallel, each in its own isolated git worktree, respecting dependencies and a parallel limit, so that runs are faster without agents interfering with each other.
- US-5: As an operator, I want every task independently tested and reviewed against the spec, with rejected work sent back to the Builder a limited number of times and then escalated to me, so that quality is checked by someone other than the author.
- US-6: As an operator, I want approved tasks merged into an integration branch and the full suite run after each merge, so that tasks that pass alone but break together are caught.
- US-7: As an operator, I want a final summary and the decision on whether to merge into my main branch, so that nothing reaches my main branch without my approval and nothing is pushed (Gate 2).
- US-8: As an operator, I want guardrails enforced by the tooling (not just requested in prompts): per-agent tool permissions, blocking of dangerous commands, budget caps, and protection of my repo and history, so that I can run Relay safely.
- US-9: As an operator, I want to resume an interrupted run, so that a crash, a kill or a usage limit doesn't cost me finished work.
- US-10: As an operator, I want a report and full transcripts after every run, so that I can see what happened, what it cost and what to improve.
- US-11: As an operator, I want to define and override agents and pipeline settings per target project, so that I can adapt Relay to different codebases (for example a stricter Reviewer).
- US-12: As an operator, I want to choose how agents run (headless Claude Code, Agent SDK or scripted) and to run a demo with no API calls, so that I can control cost and try Relay safely.
- US-13: As an operator, I want a live dashboard of the agents at work, so that I can follow progress, handoffs, rejections and spend as they happen.
- US-14: As a viewer, I want to watch a recorded run in a browser with no key or backend, so that I can see how Relay works.

## 4. Acceptance criteria
AC-1 to AC-9 are the acceptance criteria from `SPEC.md`, in their original order and wording. AC-10 onwards are taken from other statements in `SPEC.md`, cited in brackets.

**Starting a run (US-1)**
- AC-10 (US-1): Given a target repo with no uncommitted changes, when I run `relay run --spec ./SPEC.md --repo ./target-folder`, then a run ID is created and `.relay/runs/<run-id>/events.jsonl` is created in the target, starting with a "run started" event. [Pipeline 1, Events]
- AC-7 (US-1, US-10): Given a real run on the sample spec, the result builds, passes its tests and has a `REPORT.md` with cost per agent.

**Analysis (US-2)**
- AC-11 (US-2): Given a spec, when the Analyst runs, then it produces at most N questions (N from config, default 5), the CLI asks them one at a time, and the answers are saved to `.relay/answers.md`. [Pipeline 2]

**Planning and Gate 1 (US-3)**
- AC-2 (US-3): Given a task graph with a dependency cycle or an uncovered acceptance criterion, the Planner's output is rejected with a clear error and retried.
- AC-12 (US-3): Given a task graph with duplicate task IDs, a task sized L, or output that doesn't match the task-graph schema, then it is rejected with a clear error naming the problem and the errors go back to the Planner, with at most 2 retries. [Task graph]
- AC-13 (US-3): Given a valid task graph, when planning completes, then `PLAN.md` and the task graph are written to `.relay/plan/` and the CLI shows the plan with options to approve or reject. If I edit the plan file and re-run, the edited plan is validated with the same rules and used. [Pipeline 3–4]
- AC-14 (US-3): Given the plan is shown at Gate 1, when I reject it, then no Builder starts, no worktree is created and the run ends with a "gate" event recording the rejection. [Pipeline 4]

**Parallel execution (US-4)**
- AC-3 (US-4): Given `maxParallel: 2` and four independent tasks, no more than two Builders are ever active at once (verified from events).
- AC-15 (US-4): Given task B depends on task A, then no agent starts work on B until A is Done (verified from events). [Pipeline 5]
- AC-16 (US-4): Given a task with ID `<id>` starts, then its Builder works in its own git worktree on branch `relay/task-<id>`, and no other task's agent works in that worktree. [Agent team; D2]

**Test and review loop (US-5)**
- AC-1 (US-5, US-12): Given the sample spec and the FakeRunner, when I run `relay run --demo`, the whole pipeline completes and `events.jsonl` contains the full sequence, including at least one scripted rejection that is fixed in round 2.
- AC-17 (US-5): Given the Tester reports a failure, then the failure report goes back to the Builder of that task and is recorded as a verdict event and a handoff event. [Pipeline 5]
- AC-18 (US-5): Given the Reviewer returns "Request changes", then its specific, numbered points go back to the Builder and are recorded in a verdict event. [Agent team; Pipeline 5]
- AC-4 (US-5): Given a Reviewer that rejects 3 times with `maxReviewRounds: 2`, the task goes to Needs human and the run continues with other tasks.

**Integration (US-6)**
- AC-19 (US-6): Given a task approved by the Tester and Reviewer, then the Integrator merges its branch into `relay/integration` and runs the full test suite. If the suite fails, the task goes back to its Builder with the failure output. [Pipeline 5; Agent team]

**Final review, Gate 2 (US-7)**
- AC-20 (US-7): Given all tasks have finished (Done or Needs human), then the CLI shows a summary of tasks, rounds per task, cost, test results and diff stats, and asks whether to merge `relay/integration` into my main branch. Relay merges only if I approve. [Pipeline 6]
- AC-21 (US-7, US-8): Given any run, then Relay never pushes to a remote (no `git push` by the engine or any agent). [Pipeline 6; Guardrails]

**Guardrails (US-8)**
- AC-5 (US-8): Given an agent that tries `git push` or edits a file outside its worktree, the action is blocked and logged, and the run continues.
- AC-22 (US-8): Given any agent, when it tries `rm -rf` outside its worktree, piping a download into a shell (for example `curl … | sh`), editing a file outside the target repo, or reading a `.env` file (through any tool, including Bash), then the action is blocked, logged as an event, and the run continues. [Guardrails]
- AC-23 (US-8): Given the agent permissions in the roster (Analyst and Reviewer cannot edit anything; Planner can write only `.relay/plan/`; Tester can edit test files only; Builders cannot edit `SPEC.md`, other worktrees or push), when an agent tries an action outside its permissions, then it is blocked by the tool allow-list or a hook (not only by its prompt) and logged. [Agent team; Guardrails]
- AC-24 (US-8): Given a budget cap per agent call or per run (in dollars and turns), when a cap is reached, then the run pauses at a safe point (no half-applied merge) and asks me whether to continue, raise the cap or stop. [Guardrails]
- AC-25 (US-8): Given the target repo has uncommitted changes, when I start a run, then Relay refuses with a clear message and changes nothing. [Guardrails]
- AC-26 (US-8): Given any run, then Relay creates and changes only branches named `relay/*` and leaves every other branch's commits unchanged (verified by comparing refs before and after). [Guardrails]

**Resume (US-9)**
- AC-6 (US-9): Given a run killed halfway, when I run `relay resume`, it continues from the last completed step without redoing finished tasks.
- AC-27 (US-9): Given a run's `events.jsonl`, when `run.json` is rebuilt from the events alone, then it equals the saved `run.json`. [Events and state]

**Report and transcripts (US-10)**
- AC-28 (US-10): Given a finished run, then `.relay/runs/<run-id>/REPORT.md` contains the timeline, every handoff, every rejection with its reason, cost per agent, and lessons including the tasks that needed the most rounds. [Pipeline 7]
- AC-29 (US-10): Given a finished run, then the full transcript of every agent call is saved per agent per task. [Events and state]

**Configurable agents and pipeline (US-11)**
- AC-30 (US-11): Given the default agents in `agents/*.md` (frontmatter: `name`, `role`, `model`, `allowedTools`, `maxTurns`, `maxBudgetUsd`; body: system prompt), when the target has `.relay/agents/reviewer.md`, then that file replaces the default Reviewer for that run, and a new file there adds a new agent definition. [Agent definitions]
- AC-31 (US-11): Given a `relay.config.json` in the target, then its values override Relay's defaults for stage order, gates on or off, `maxParallel`, `maxReviewRounds`, budget caps, and the test, lint and build commands. [Agent definitions]
- AC-32 (US-11): Given the target has a `CLAUDE.md`, then every agent's prompt tells it to read that file. [Agent definitions]

**Runners and demo mode (US-12)**
- AC-33 (US-12): Given the same scripted agent interaction, when it passes through the FakeRunner and through the parsers of the ClaudeCodeRunner and AgentSdkRunner (using recorded output), then all three emit the same sequence of normalised event types. [Runners]
- AC-34 (US-12): Given no runner is set in config, then Relay uses the ClaudeCodeRunner (headless Claude Code with the operator's login). The AgentSdkRunner is used only when config selects it. [Decisions to make; D1]
- AC-9 (US-12): `npm test` makes zero network calls.

**Live dashboard (US-13)**
- AC-35 (US-13): Given a run in progress, when I run `relay ui`, then a local web app opens and updates live through server-sent events as events are appended. [Dashboard]
- AC-36 (US-13): Given the swimlane view, then there is one lane per agent instance, blocks show what each is doing, handoffs are drawn as arrows between lanes, and rejections are drawn as red return arrows labelled with the reason. [Dashboard]
- AC-37 (US-13): Given the task board, then it has the columns Queued, Building, Testing, In review, Integrating, Done and Needs human, and each card shows its round count. [Dashboard]
- AC-38 (US-13): Given I click any block, then the detail pane shows the prompt summary, tool calls, diff and verdict for that block. [Dashboard]
- AC-39 (US-13): Given the dashboard, then the meters show cost so far against budget, tokens per agent, elapsed time, and tasks done out of the total. [Dashboard]

**Replay (US-14)**
- AC-40 (US-14): Given any `events.jsonl`, when I load it in replay mode, then I can play, pause, change speed and scrub through the run. [Dashboard]
- AC-41 (US-14): Given a recorded run, when I run `npm run build:replay -- <run-id>`, then a static site with that run built in is produced. [Dashboard]
- AC-8 (US-14): The replay site plays a recorded run on GitHub Pages with no backend.

## 5. Non-functional requirements
- NFR-1 **Orchestrator is code, not an agent:** the engine that decides who works next is plain TypeScript with predictable, testable control flow. Agents never choose the workflow.
- NFR-2 **Testability:** all core logic is testable with the FakeRunner. Core tests cover scheduling, dependency order, parallel limits, rejection loops, budget pauses, resume and graph validation. Tests never call an API or the network (AC-9).
- NFR-3 **Events:** every action appends one JSON line to `events.jsonl`: run started, agent started, tool used, message, handoff, verdict, gate, merge, cost update, error, run finished. Each event has a timestamp, agent, task ID and payload. Events are append-only.
- NFR-4 **Guardrails are enforced by the tools:** permissions use tool allow-lists, deny rules and hooks, never prompt instructions alone (AC-5, AC-22, AC-23).
- NFR-5 **Cost figures are estimates:** on the ClaudeCodeRunner (subscription) the dollar figures are an estimated equivalent API cost, and reports and the dashboard label them that way. Budget caps use the same estimate.
- NFR-6 **Usage limits:** when the subscription's usage limit is hit (a rate-limit retry from the CLI), the run pauses at a safe point and tells the operator, and `relay resume` continues later (AC-6). [D7]
- NFR-7 **Tech:** Node 20+, TypeScript, npm workspaces monorepo: `packages/core` (engine, graph, events, runners), `packages/cli`, `packages/dashboard` (Vite, React, TypeScript, plain CSS). Vitest for tests. zod (or similar) for task-graph validation. `fixtures/sample-spec/` holds a tiny spec (a tip calculator) for smoke runs.
- NFR-8 **Platform:** runs natively on Windows (PowerShell), including paths with spaces. CI (GitHub Actions) runs tests on Ubuntu and deploys the static replay site to GitHub Pages. [D4]
- NFR-9 **README:** a one-paragraph pitch; a GIF of the swimlane dashboard showing a rejection loop; a link to the live replay; an architecture diagram of the pipeline and gates; the agent roster table; the guardrails; how to define your own agents; "Built X with Relay" with numbers (tasks, rounds, cost, time); honest lessons; and a "How this was built with Claude Code" section linking to `docs/PLAN.md` and `docs/DECISIONS.md`.

## 6. Agent roster (defaults)
| Agent | Job | Can | Cannot | Default model |
|---|---|---|---|---|
| Analyst | Lists ambiguities and missing acceptance criteria as questions | Read files | Edit anything | haiku |
| Planner | Turns spec and answers into a task graph and `PLAN.md` | Read files, write `.relay/plan/` | Edit source code | opus |
| Builder (one per task in progress) | Implements one task in its worktree, tests first | Read, edit, run allowed commands in its worktree | Edit `SPEC.md`, touch other worktrees, push | sonnet |
| Tester | Checks the task's acceptance criteria independently; writes or extends tests; runs the full suite; reports pass or fail with evidence | Edit test files, run tests | Edit non-test source files | haiku |
| Reviewer | Reviews the diff against the spec, the task's criteria and project rules; returns Approve or Request changes with numbered points | Read files and diffs | Edit anything | sonnet |
| Integrator | Merges approved branches into `relay/integration`, runs build and tests, resolves simple conflicts or sends the task back | Git merge in the main repo, run commands | Push to a remote | sonnet |

## 7. Out of scope
- Pushing to remotes or opening pull requests (stretch goal only)
- Non-Claude models
- A hosted multi-user service
- Agents choosing their own workflow (the orchestrator decides the flow)

## 8. Questions
**Resolved (operator, 2026-11-22; see `docs/DECISIONS.md`)**
- Default runner: ClaudeCodeRunner (headless, Pro subscription). AgentSdkRunner is built but opt-in. See AC-34. (D1)
- Worktrees: one per task. See AC-16. (D2)
- Models: Planner opus; Analyst and Tester haiku; Builder, Reviewer and Integrator sonnet. See section 6. (D3)
- Platform: native Windows PowerShell; CI on Ubuntu. See NFR-8. (D4)
- Defaults: Analyst at most 5 questions; caps of $3 per call, $25 per run (estimated), 40 turns per call; `maxParallel` 2; `maxReviewRounds` 2. (D5)

**Resolved (operator, 2026-11-23; all proposed answers accepted; see `docs/DECISIONS.md` D9-D14)**
- Q1. **Planner still invalid after 2 retries.** The run stops before Gate 1 and shows the validation errors. Nothing is built. (D9)
- Q2. **Options when a task reaches Needs human.** (a) retry with a note from me to the Builder, which resets the round count once; (b) skip the task; (c) stop the run. (D10)
- Q3. **Tasks that depend on a Needs-human task.** They stay blocked (not started) and are listed in the Gate 2 summary. (D11)
- Q4. **What `maxParallel` limits.** The number of tasks in progress at once, so at most `maxParallel` Builders, Testers and Reviewers in total across tasks. This satisfies AC-3. (D12)
- Q5. **"Simple conflicts" for the Integrator.** The Integrator may resolve a conflict only by editing the conflicting sections, and only if the full suite then passes. Otherwise the task goes back to its Builder with the conflict. (D13)
- Q6. **No ambiguities found by the Analyst.** No questions are asked, an empty `answers.md` is written, and planning starts. (D14)
