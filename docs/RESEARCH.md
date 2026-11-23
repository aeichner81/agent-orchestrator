# Research: Claude Agent SDK (TypeScript) and Claude Code headless mode

Checked against the live docs on 21 Nov 2026. Re-check before Phase 2, because the CLI and SDK version quickly.

## 1. Two ways to run an agent

| | Agent SDK (`@anthropic-ai/claude-agent-sdk`) | Claude Code headless (`claude -p`) |
|---|---|---|
| What it is | TypeScript library. It bundles a native Claude Code binary and drives it for you. | The Claude Code CLI run non-interactively as a child process. |
| Auth | API key (`ANTHROPIC_API_KEY`), or Bedrock/Vertex/Foundry. The docs say third-party products must not offer claude.ai login unless Anthropic has approved it. | Your existing Claude Code login (subscription), or an API key. |
| Billing | Pay as you go on API prices. Since 7 Oct 2026, Max, Ultra and Team plans include monthly API credits that also cover SDK use with a key from a linked Console org. | With a plan login, usage comes out of your subscription limits. A separate SDK credit pool was announced for 15 Jun 2026 and then **paused**, so for now nothing has changed. |
| Results | Typed `SDKMessage` objects | Newline-delimited JSON (`--output-format stream-json`) |
| Hooks | In-process TypeScript callbacks (`options.hooks`) | Shell-command hooks passed in with `--settings <json>` |

Sources:
- SDK overview: https://code.claude.com/docs/en/agent-sdk/overview
- Headless mode: https://code.claude.com/docs/en/headless
- CLI flags: https://code.claude.com/docs/en/cli-reference
- Plan billing: https://support.claude.com/en/articles/15036540

## 2. Running a query

**SDK:** `query({ prompt, options }): Query`. It returns an async iterator of `SDKMessage`. `prompt` can be a string or an `AsyncIterable<SDKUserMessage>`, and the async-iterable form is needed for streaming input and for changing the permission mode mid-run.
Docs: https://code.claude.com/docs/en/agent-sdk/typescript

**CLI:** `claude -p "<prompt>" --output-format stream-json --verbose [--include-partial-messages]`. The exit code is 0 on success and non-zero on failure. A failure inside the run, such as missing auth, is printed as the result on stdout.

## 3. Streaming messages and events

- The stream starts with `system/init` (model, tools, MCP servers, plugins, `capabilities`). `SessionStart` or `Setup` hook events can come before it.
- Then `assistant` and `user` messages, which carry the tool_use and tool_result blocks. Subagent messages have a non-null `parent_tool_use_id`.
- `stream_event` lines appear only with `--include-partial-messages` (token deltas).
- `system/api_retry` is sent before a retry. Its `error` field can be `rate_limit`, `overloaded`, `billing_error`, `authentication_failed` and so on. **This is how Relay will notice subscription limits and pause.**
- `permission_denied` system messages appear with stream-json, and the result lists them in `permission_denials`.
- The final line is always a `result` message.

Relay's normalised events (SPEC "Events") will be mapped from these.

## 4. Tool allow-lists and permissions

| Need | SDK option | CLI flag |
|---|---|---|
| Limit which built-in tools exist at all | `tools` | `--tools "Read,Grep"` (`""` = none) |
| Auto-approve tools. **Not a restriction:** tools not listed fall through to the permission mode | `allowedTools` | `--allowedTools` |
| Remove a tool, or deny matching calls | `disallowedTools`. A scoped rule such as `Bash(git push *)` is denied in **every** mode, including `bypassPermissions` | `--disallowedTools` |
| Baseline mode | `permissionMode` | `--permission-mode default\|acceptEdits\|plan\|auto\|dontAsk\|bypassPermissions` |
| Decide prompts in code | `canUseTool` callback | `--permission-prompt-tool` (MCP tool) |
| Nobody available to answer prompts | (no prompt host) | `--permission-prompts none` (v2.1.259+) |

Rule syntax is `Bash(git diff *)`. The space before `*` matters.
Since v2.1.302, `claude -p` starts in Manual mode, so any call that needs approval is denied unless something approves it.
**Implication for Relay:** each agent gets `tools` (what exists) plus `allowedTools` (what is auto-approved) plus `disallowedTools` (hard denies), and runs in `dontAsk` mode. Anything else is denied without hanging the run.
Docs: https://code.claude.com/docs/en/agent-sdk/permissions

## 5. Hooks (the guardrails)

- SDK: `hooks: { PreToolUse: [{ matcher: "Bash|Write|Edit", hooks: [cb], timeout }] }`.
- Callback: `(input, toolUseID, { signal }) => Promise<HookJSONOutput>`.
- To block a call, return `{ hookSpecificOutput: { hookEventName: "PreToolUse", permissionDecision: "deny", permissionDecisionReason } }`. Returning `{}` allows it.
- When several hooks run, **deny wins**. If a PreToolUse hook times out, the tool call is blocked.
- Hooks also fire inside subagents (`agent_id` and `agent_type` are on the input).
- Matchers only match tool names, so path checks have to be done inside the hook. A `Write|Edit` matcher does **not** cover writes made through Bash, so the Bash command must be checked as well.
- CLI equivalent: the same JSON contract from a shell-command hook defined in a `--settings` JSON. Relay will ship the guard as one Node script (works on Windows and Linux) and call it from both runners, so there is one implementation with one set of tests.
- Other useful events: `PostToolUse` (audit log), `SubagentStart`/`SubagentStop`, `Stop`, `SessionEnd`, `WorktreeCreate`/`WorktreeRemove` (TypeScript only).

Docs: https://code.claude.com/docs/en/agent-sdk/hooks and https://code.claude.com/docs/en/hooks

## 6. Budgets and turn limits

- `maxTurns` / `--max-turns`: the run exits with an error when the limit is reached.
- `maxBudgetUsd` / `--max-budget-usd`: stops when the **client-side estimate** reaches the cap. Subagent spend counts towards it, and spend can go slightly over.
- On the subscription runner these dollars are an "equivalent API cost" estimate, not money spent. Relay will label them that way.

## 7. Cost and usage reporting

- The `result` message has `total_cost_usd`, cumulative `usage` (including `cache_creation_input_tokens` and `cache_read_input_tokens`), `modelUsage` (per model, counting the whole subagent tree; use this rather than `usage`, which undercounts when subagents nest), `num_turns`, `duration_ms`, `session_id` and `subtype`.
- Both success and error results include cost, so always read it.
- Costs are estimates from a price table bundled with the SDK. The authoritative figure is the Console Usage page or the Usage and Cost API.
- In the CLI, `--resume` reports the whole conversation's total including earlier runs. Relay must store deltas, not totals.

Docs: https://code.claude.com/docs/en/agent-sdk/cost-tracking

## 8. Structured output (Planner's task graph)

- SDK: `outputFormat: { type: "json_schema", schema }`. CLI: `--output-format json --json-schema '<schema>'`. The result is in `structured_output`.
- If the field is missing (for example, the model declined), treat that as a validation failure and use one of the Planner's two retries.
- Relay still validates with zod afterwards: no cycles, all acceptance criteria covered, no tasks sized L. JSON Schema can't express those rules.

## 9. Session resume

- SDK: `resume: sessionId`, `forkSession`, `continue`. CLI: `--resume <id|path-to-jsonl>`, `--fork-session`, `--session-id <uuid>`, `--continue`.
- On SIGTERM the CLI exits with 143 and the turn in progress is left unfinished. To end a turn cleanly, send SIGINT or call `interrupt()`.
- Relay's `relay resume` rebuilds state from `events.jsonl` (SPEC) and re-runs only incomplete agent steps. Agent session IDs are stored in events so an interrupted agent call can be resumed instead of restarted.

## 10. Isolation and reproducibility

- `--bare` skips the user's hooks, plugins, MCP servers, CLAUDE.md and auto memory. It is recommended for scripts and will become the default for `-p`. **But `--bare` never reads OAuth credentials, so it needs an API key.**
- So the subscription runner cannot use `--bare`. Instead it will pass `--setting-sources project`, `--strict-mcp-config` (with an empty config), an explicit `--tools` list and an explicit `--system-prompt`. That keeps out personal settings from `~/.claude`.
- Note: without `--bare`, `-p` runs the target project's `.claude/settings.json` hooks and `.mcp.json` servers **without a trust prompt**. Relay should warn if the target repo has them.
- SDK: `settingSources: []` turns off user, project and local settings. `env` *replaces* the subprocess environment, so spread `process.env`.

## 11. Windows notes (operator is on Windows)

- The SDK binary on Windows is `claude.exe`. `pathToClaudeCodeExecutable` can point to an installed `claude`.
- Hook scripts will be Node, not bash, so they behave the same on Windows and in GitHub Actions (Ubuntu).
- On Windows, `file_path` uses backslashes, so the guard normalises paths before checking whether they're inside the worktree.
