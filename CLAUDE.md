# Project rules: Agent Relay

## Source of truth
- `specs/001-agent-relay.md` is the spec. Use its IDs (US-x, AC-x, NFR-x, Q-x). `SPEC.md` is the original brief, kept for reference.
- `docs/PLAN.md` is the approved phase plan. `docs/RESEARCH.md` holds the verified Agent SDK and headless-CLI API notes. `docs/DECISIONS.md` is the decisions log.
- If the spec, plan and code disagree, stop and ask. Don't choose silently.

## Workflow
- Don't add features that aren't in the spec. Suggest additions and wait for agreement.
- Work one phase at a time, following `docs/PLAN.md`. Plan in plan mode before each phase.
- Every task names the AC IDs it satisfies. Write tests first for those ACs, then the code.
- All core logic must be testable with the FakeRunner. No API or network calls in tests, ever (AC-9).
- After each phase: run tests, lint and build, then stop and summarise what changed so the owner can review and commit. Don't commit or push unless asked.
- Record key decisions in `docs/DECISIONS.md` as they're made.
- Don't rely on memory for Claude Agent SDK or Claude Code CLI APIs. Check `docs/RESEARCH.md` and the live docs.

## Tech
- Node 20+, TypeScript (strict), npm workspaces: `packages/core`, `packages/cli`, `packages/dashboard` (Vite + React + plain CSS). Vitest. zod.
- Runs natively on Windows PowerShell, and paths may contain spaces. CI runs on Ubuntu. Write scripts and hooks in Node, not bash.

## Defaults agreed with the owner
- Default runner: ClaudeCodeRunner (headless `claude -p`, Pro subscription). The AgentSdkRunner is opt-in only, because it bills per token.
- One git worktree per task. Models: Planner opus; Analyst and Tester haiku; Builder, Reviewer and Integrator sonnet.
- maxParallel 2, maxReviewRounds 2, at most 5 Analyst questions, caps of $3 per call and $25 per run (estimated), 40 turns per call.
