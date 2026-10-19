# Research: Claude Code headless capabilities

- Ticket: [jim-at-jibba/agent-os#11](https://github.com/jim-at-jibba/agent-os/issues/11)
- Date: 2026-10-19
- Local CLI checked: `claude --version` -> `2.1.303 (Claude Code)`; `claude --help` output used as the primary source for flag names and recorded below where it differs from the docs.
- Doc hosts: the Claude Code docs now live at `code.claude.com/docs` (the `docs.claude.com/en/docs/claude-code/...` paths redirect there), and the Claude API / prompting docs live at `platform.claude.com/docs`. Both are first-party Anthropic documentation. Every claim below links to the page it came from.

## TL;DR

- Headless = `claude -p "<prompt>"` (alias `--print`). Add `--bare` for CI (skips hooks, plugins, MCP, CLAUDE.md, OAuth). Exit code 0 on success, non-zero on failure.
- Output: `--output-format text|json|stream-json`. `json` returns one object with `result`, `session_id`, `total_cost_usd`, `usage`, `modelUsage`, `num_turns`, `duration_ms`, `permission_denials`, `is_error`, `subtype`. `--json-schema '<schema>'` adds a validated `structured_output` field.
- Model: `--model <alias|full-id>` (aliases `sonnet`, `opus`, `haiku`, `fable`, `best`, `opusplan`, `sonnet[1m]`, `opus[1m]`); `--fallback-model a,b`. Effort: `--effort low|medium|high|xhigh|max` (docs also accept `ultracode`); `CLAUDE_CODE_EFFORT_LEVEL` env var overrides the flag.
- Permissions: `--permission-mode default|manual|acceptEdits|plan|auto|dontAsk|bypassPermissions`; `--allowedTools` / `--disallowedTools` (rule syntax `Bash(git diff *)`, `Edit`, `Read(./.env)`); `--tools` restricts the built-in tool set; `--permission-prompts none` denies anything that would prompt; `--restricted` strips command-running tools; `--dangerously-skip-permissions` = bypass mode. Sandboxing is a settings object (`sandbox.enabled`, `sandbox.network.allowedDomains`, ...), not a flag, and can be injected with `--settings '{"sandbox":{...}}'`.
- Cost: `--output-format json` carries `total_cost_usd` (client-side estimate) plus `usage` and per-model `modelUsage[model].costUSD`. Interactive `/usage` and OpenTelemetry cover the non-headless cases.
- Budget: `--max-budget-usd <amount>` (print mode only; result `subtype: "error_max_budget_usd"`) and `--max-turns <n>` (result `subtype: "error_max_turns"`). The Messages API "task budget" (`output_config.task_budget`, beta header `task-budgets-2026-03-13`) is a separate, advisory token budget; the task-budgets page says it is **not supported on Claude Code surfaces**, while the Python Agent SDK reference documents a `task_budget` option. See section 6.
- Anti-fabrication instruction: Anthropic's "Ground progress claims during long runs" prompt, quoted verbatim in section 7.

## 1. Running headless (print mode)

Source: [Run Claude Code programmatically](https://code.claude.com/docs/en/headless) and [CLI reference](https://code.claude.com/docs/en/cli-reference).

| Flag | What it does |
| --- | --- |
| `-p`, `--print` | "Print response and exit (useful for pipes)." Starts non-interactive mode. The workspace-trust dialog is skipped, and settings files that fail validation are silently ignored (local `--help`). |
| `--bare` | "Minimal mode: skip hooks ..., LSP, plugin sync, attribution, auto-memory, background prefetches, keychain reads, and CLAUDE.md auto-discovery. Sets `CLAUDE_CODE_SIMPLE=1`." Auth is strictly `ANTHROPIC_API_KEY` or `apiKeyHelper` via `--settings` (OAuth/keychain never read). The docs say `--bare` "is the recommended mode for scripted and SDK calls, and will become the default for `-p` in a future release." |
| `--input-format text\|stream-json` | Print mode only. `stream-json` enables realtime streaming input. |
| `--no-session-persistence` | Print mode only; the session is not saved to disk and cannot be resumed. |
| `--continue` / `--resume <id\|name\|transcript-path>` | Continue the last conversation or a specific one. `claude -p "..." --output-format json \| jq -r '.session_id'` captures the ID to resume. |
| `--session-id <uuid>` | Pin the session ID. |
| `--system-prompt`, `--system-prompt-file`, `--append-system-prompt`, `--append-system-prompt-file` | Replace or append to the system prompt. `--append-subagent-system-prompt[-file]` does the same for subagents (print mode only). |
| `--verbose` | Full turn-by-turn output; required for `stream-json` partial messages and some stream-only flags. |
| `--max-turns <n>` | "Limit the number of agentic turns (print mode only). Exits with an error when the limit is reached. No limit by default." **Not shown in local `claude --help` 2.1.303**, but the CLI reference states "`claude --help` does not list every flag". |
| `--max-budget-usd <amount>` | See section 6. |

Behaviour notes from the headless page:

- "Claude Code exits with code 0 on success and a non-zero code when the run fails." Invalid flags go to stderr before the run; failures inside the run (e.g. missing auth) are printed as the result on stdout.
- Without `--bare`, a `-p` session runs the hooks in the project's `.claude/settings.json` and connects servers in `.mcp.json` "even in a folder you've never trusted".
- Piped stdin is capped at 10 MB.
- After the final turn, a `-p` run waits for background work (background commands, subagents, Monitor watches) up to a 10-minute idle ceiling (`CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`; `0` waits indefinitely).
- SIGTERM exits with code 143 and leaves the in-progress turn unfinished; send SIGINT to end the turn first.
- `/skill-name`, `/model sonnet`, `/effort high`, `/config key=value` all work inside the `-p` prompt string.
- `-p` cannot be combined with `--bg`.

## 2. Output formats and the JSON result object

Source: [headless page, "Get structured output"](https://code.claude.com/docs/en/headless#get-structured-output); [TypeScript SDK reference, `SDKResultMessage`](https://code.claude.com/docs/en/agent-sdk/typescript#sdkresultmessage); [agent loop, "Handle the result"](https://code.claude.com/docs/en/agent-sdk/agent-loop#handle-the-result).

`--output-format` (print mode only) accepts:

- `text` (default): plain text.
- `json`: "structured JSON with result, session ID, and metadata". Single object on stdout.
- `stream-json`: newline-delimited JSON events; "the last line of the stream is a `result` message with the final response text, cost, and session metadata". Add `--verbose --include-partial-messages` for token deltas, `--include-hook-events` for hook lifecycle, `--forward-subagent-text` to get subagent text/thinking blocks (each tagged with `parent_tool_use_id`).

`--json-schema '<JSON Schema>'` with `--output-format json` makes Claude Code validate the output; the validated value lands in `structured_output`. An invalid schema exits with `Error: --json-schema is not a valid JSON Schema`. The `format` keyword is accepted but not enforced.

The `json` result object is the SDK's `SDKResultMessage`. Fields common to the success and error variants (verbatim from the TypeScript reference):

```
type: "result"
subtype: "success" | "error_max_turns" | "error_during_execution" | "error_max_budget_usd" | "error_max_structured_output_retries"
uuid, session_id
duration_ms, duration_api_ms
is_error: boolean
num_turns: number
stop_reason: string | null        // "end_turn", "max_tokens", "refusal", ...
total_cost_usd: number
usage: NonNullableUsage           // input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens, ...
modelUsage: { [modelName]: ModelUsage }
permission_denials: SDKPermissionDenial[]
```

Success-only: `result: string`, `structured_output?: unknown`. Error-only: `errors: string[]`. `result` is "only present on the `success` variant, so always check the subtype before reading it."

`ModelUsage` (per model): `inputTokens`, `outputTokens`, `thinkingTokens?`, `cacheReadInputTokens`, `cacheCreationInputTokens`, `webSearchRequests`, `costUSD`, `contextWindow`, `maxOutputTokens`, `canonicalModel?`, `provider?`, `costBasis?: 'list' | 'managed' | 'unknown'`.

Example from the docs:

```bash
claude -p "Summarize this project" --output-format json | jq -r '.result'
claude -p "Extract the main function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  | jq '.structured_output'
```

Stream-only event types worth knowing: `system/init` (model, tools, `mcp_servers`, `plugins`, `plugin_errors`, `mcp_server_errors`), `system/api_retry`, `permission_denied`.

## 3. Selecting model and effort

Source: [Model configuration](https://code.claude.com/docs/en/model-config); [CLI reference](https://code.claude.com/docs/en/cli-reference); [Environment variables](https://code.claude.com/docs/en/env-vars); [agent loop, "Effort level"](https://code.claude.com/docs/en/agent-sdk/agent-loop#effort-level).

### Model

Priority order documented on the model-config page: `/model` in-session (also works with `-p`, for that session only) > `--model` flag > `ANTHROPIC_MODEL` env var > `model` setting in settings.json > `ANTHROPIC_DEFAULT_MODEL` (v2.1.236+).

- `--model <model>`: "Provide an alias for the latest model (e.g. 'fable', 'opus', or 'sonnet') or a model's full name" (local `--help`).
- Aliases: `default`, `best`, `fable`, `sonnet`, `opus`, `haiku`, `sonnet[1m]`, `opus[1m]`, `opusplan` (Opus in plan mode, Sonnet for execution).
- Pinning: use the full ID (e.g. `claude-opus-5-5`) or set `ANTHROPIC_DEFAULT_OPUS_MODEL` / `ANTHROPIC_DEFAULT_SONNET_MODEL` / `ANTHROPIC_DEFAULT_HAIKU_MODEL` / `ANTHROPIC_DEFAULT_FABLE_MODEL`.
- `--fallback-model <a,b>`: comma-separated chain tried in order when the primary is overloaded/unavailable; capped at three; the switch lasts one turn; the `fallbackModel` setting persists it.

### Effort

- `--effort <level>`: local `--help` lists `low, medium, high, xhigh, max`. The CLI reference adds `ultracode` ("requests `xhigh` effort with ultracode turned on", v2.1.203+). "Overrides the `modelSettings` and `effortLevel` settings for this session and does not persist."
- `CLAUDE_CODE_EFFORT_LEVEL`: "Takes precedence over `--effort`, `/effort`, and the `modelSettings` and `effortLevel` settings. A `maxEffortLevel` cap still applies." Accepts a level or `auto`.
- `effortLevel` setting accepts `low|medium|high|xhigh` (not `max`); `modelSettings` stores a per-model level; skill/subagent frontmatter `effort` overrides while that runs.
- Supported levels by model (model-config page): Fable 5/5.1, Opus 5/5.5, Sonnet 5/5.5, Haiku 5.5, Opus 4.7/4.8: `low..max`; Opus 4.6 / Sonnet 4.6: `low, medium, high, max`. Unsupported levels round down to the highest supported level at or below.
- Defaults: `high` on most models; `medium` on Opus 5.5, Sonnet 5.5, Haiku 5.5; `xhigh` on Opus 4.7.
- `MAX_THINKING_TOKENS` only affects fixed-thinking-budget models; adaptive-reasoning models ignore it, so use effort there.

## 4. Restricting permissions and tools

Sources: [Permission modes](https://code.claude.com/docs/en/permission-modes); [Configure permissions](https://code.claude.com/docs/en/permissions); [CLI reference](https://code.claude.com/docs/en/cli-reference); [headless, "Auto-approve tools"](https://code.claude.com/docs/en/headless#auto-approve-tools) and ["Turn off permission prompts in unattended runs"](https://code.claude.com/docs/en/headless#turn-off-permission-prompts-in-unattended-runs); [Sandboxing](https://code.claude.com/docs/en/sandboxing).

### Permission modes (`--permission-mode <mode>`)

Local `--help` choices: `acceptEdits, auto, bypassPermissions, manual, dontAsk, plan`. The docs: "Accepts `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`, or `manual` as an alias for `default`" (`manual` alias needs v2.1.200+). Overrides `permissions.defaultMode` from settings.

| Mode | What runs without asking (docs table) |
| --- | --- |
| `default` / `manual` | Reads only |
| `acceptEdits` | Reads, file edits, common filesystem commands (`mkdir`, `touch`, `mv`, `cp`, ...) |
| `plan` | Reads, plus classifier-approved commands when auto mode is available; "does not edit your source" |
| `auto` | Everything, with a server-side classifier reviewing actions |
| `dontAsk` | File reads inside working dirs and pre-approved tools; "anything that would prompt is denied" - "Locked-down CI and scripts" |
| `bypassPermissions` | Everything - "Isolated containers and VMs only" |

What a `-p` run starts in when no flag/setting applies: "`default` in sessions that fetch feature flags. In sessions that don't, such as on a third-party provider or with telemetry off, `auto` with Claude Code v2.1.285 or later and `default` on earlier versions." The headless page therefore says: "pass the one you want".

Plan mode in headless: "Plan mode keeps its blocks wherever Claude Code runs without an interactive terminal, including non-interactive runs with `-p`, Agent SDK sessions ...". Start with `claude --permission-mode plan`. (In interactive terminals plan mode is exited by approving the plan or `Shift+Tab`; there is no documented headless "approve plan" step - see section 9.)

Bypass: `--dangerously-skip-permissions` is "Equivalent to `--permission-mode bypassPermissions`". `--allow-dangerously-skip-permissions` only adds bypass to the mode cycle without starting in it. Refused when running as root/sudo on Linux/macOS (skipped inside a recognized sandbox). Admins block it with `permissions.disableBypassPermissionsMode: "disable"` and auto mode with `permissions.disableAutoMode: "disable"` in managed settings. Deny rules still apply in every mode including bypass; allow rules have no effect in bypass. "Actions no mode auto-approves": explicit `ask` rules, `AskUserQuestion`, MCP tools marked `requiresUserInteraction`, `rm`/`rmdir` on critical paths, cross-session messaging safeguards.

### Prompts in unattended runs

- `--permission-prompts host|none` (v2.1.259+, print mode): with `none`, "anything that would prompt is denied unless a `PermissionRequest` hook allows it, Claude is told that nobody can approve the request and not to retry it, and the run continues." It also removes `AskUserQuestion`. Denials appear as `permission_denied` stream messages and in the result's `permission_denials`.
- `--permission-prompt-tool <mcp_tool>`: route prompts to an MCP tool instead.

### Allow / deny rules and tool sets

- `--allowedTools`, `--allowed-tools <tools...>`: "Tools that execute without prompting for permission". Uses permission rule syntax; e.g. `--allowedTools "Bash(git diff *),Bash(git log *),Read"`. The trailing ` *` (space before the star) enables prefix matching; `Bash(git diff*)` would also match `git diff-index`. In `auto` mode a bare `Bash` entry is dropped and the classifier evaluates each command.
- `--disallowedTools`, `--disallowed-tools <tools...>`: deny rules. "A bare tool name removes the matching tools from Claude's context: `"Edit"` removes Edit, `"*"` removes every tool, and `"mcp__*"` removes every MCP tool. A scoped rule such as `Bash(rm *)` leaves the tool available and denies only calls that match."
- `--tools <tools...>`: "Restrict which built-in tools Claude can use. Use `""` to disable all, `"default"` for the default set, or tool names like `"Bash,Edit,Read"`." Does not affect MCP tools (use `--disallowedTools "mcp__*"`).
- Rule syntax examples from the permissions page: `Bash(git commit *)`, `Read(./.env)`, `Read(./secrets/**)`, `Edit(docs/**)`, `WebFetch(domain:example.com)`, `mcp__puppeteer__*`. Path rules are only checked against `Read(...)` and `Edit(...)`; `Write(...)`/`Glob(...)` path rules are accepted but never consulted.
- Settings equivalents: `permissions.allow`, `permissions.ask`, `permissions.deny`, `permissions.additionalDirectories`, `permissions.defaultMode`; pass them inline with `--settings '<json>'` or a file, and limit which settings files load with `--setting-sources user,project,local`.
- `--strict-mcp-config`: only MCP servers from `--mcp-config`.
- `--restricted` (v2.1.248+): "removes the built-in tools that run commands or code (Bash, PowerShell and the other code-running tools) and WebFetch unless `--tools` names them, and ignores user, project and local settings files (managed settings and `--settings` still apply) ... confines the file tools to the working directories ... refuses `bypassPermissions`". Intended for "an evaluation harness [that] drives `claude` on a shared machine".
- `--safe-mode`: disables all customisations (CLAUDE.md, skills, plugins, hooks, MCP) for troubleshooting; permissions still work normally.
- `--add-dir <dirs...>`: extra directories the file tools may touch.

### Sandboxing (OS-enforced, settings-driven)

The Bash sandbox is "a boundary that the operating system enforces around the shell commands Claude runs" (macOS Seatbelt; Linux/WSL2 via `bubblewrap` + `socat`; not on native Windows). It covers Bash/PowerShell/Monitor commands only; Read/Edit/WebFetch/MCP/hooks follow permission rules instead.

- Enable: `/sandbox` in-session, or settings `"sandbox": {"enabled": true}`. Headless example from the docs: `claude --settings '{"sandbox": {"enabled": true, "allowUnsandboxedCommands": false}}'`.
- Keys: `sandbox.enabled`, `sandbox.autoAllowBashIfSandboxed` (default `true`: sandboxed commands run without prompts), `sandbox.allowUnsandboxedCommands` (`false` = strict mode, no unsandboxed retry), `sandbox.failIfUnavailable`, `sandbox.excludedCommands`, `sandbox.filesystem.allowWrite|denyWrite|allowRead|denyRead|disabled`, `sandbox.network.allowedDomains|deniedDomains|strictAllowlist|allowManagedDomainsOnly`, `sandbox.credentials` (mask env vars / files).
- Network: "No direct route out. Connections go through a proxy on your machine that checks each host against your allowed domains, which start empty." Hosts outside the allowlist prompt in asking modes; with `strictAllowlist: true` they are denied instead (needed for unattended runs).
- In `bypassPermissions` mode hosts outside the allowlist are allowed unless `strictAllowlist` or `allowManagedDomainsOnly` is on.
- Related, separate isolation options: [Sandbox environments](https://code.claude.com/docs/en/sandbox-environments) (dev containers, VMs) and cloud sessions.

## 5. Reporting cost and token usage per run

Sources: [Agent SDK cost tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking); [Manage costs](https://code.claude.com/docs/en/costs); [headless, "Pipe data through Claude"](https://code.claude.com/docs/en/headless#pipe-data-through-claude).

- Headless: "With `--output-format json`, the response payload includes `total_cost_usd` and a per-model cost breakdown, so scripted callers can track spend without consulting the usage dashboard." With `--continue`/`--resume` "the run reports the conversation's whole total, earlier runs' spend included" (v2.1.277+).
- Fields: `total_cost_usd` (includes subagents), `usage` (top-level loop only; "undercounts as soon as nesting occurs"), `modelUsage[model]` (whole tree, per model, with `costUSD` and `costBasis`), `duration_api_ms`, `num_turns`. Per-step assistant messages carry a placeholder `output_tokens`; read output tokens from the result.
- Caveat (docs warning, verbatim): "The `total_cost_usd` and `costUSD` fields are client-side estimates, not authoritative billing data." They come from a bundled price table (or an org `modelPricing` managed setting). Data-residency 1.1x pricing is modelled; web-search fees are not multiplied. For billing use the Usage and Cost API or the Console.
- Error results (`error_max_turns`, `error_max_budget_usd`, ...) still carry `total_cost_usd`, `usage`, `num_turns`, `session_id`. A session crash yields `error_during_execution` with possibly zeroed cost fields.
- Interactive-only: `/usage` prints `Total cost`, API/wall durations, and `Usage by model` (input/output/cache read/cache write tokens, $ per model); `/insights`; status-line cost field.
- Org/fleet: OpenTelemetry (`CLAUDE_CODE_ENABLE_TELEMETRY=1`) "is the only option that streams per-user token and cost metrics into your own observability stack in near real time"; Console workspace spend limits; Claude Code Analytics API; `modelPricing` managed setting for contracted rates.

## 6. Budgets: max spend, max turns, task budgets

Sources: [CLI reference](https://code.claude.com/docs/en/cli-reference#cli-flags); [agent loop, "Turns and budget" / "Budget headroom"](https://code.claude.com/docs/en/agent-sdk/agent-loop#turns-and-budget); [Task budgets (Messages API)](https://platform.claude.com/docs/en/build-with-claude/task-budgets); [Python SDK reference](https://code.claude.com/docs/en/agent-sdk/python); [Subagents, cap depth/concurrency/spend](https://code.claude.com/docs/en/agent-sdk/subagents#cap-subagent-depth-concurrency-and-spend).

### `--max-budget-usd <amount>` (dollar cap)

Local `--help`: "Maximum dollar amount to spend on API calls (only works with --print)". CLI reference: "Stop the run once estimated spend on API calls reaches this amount (print mode only). Claude Code checks the cap against its client-side cost estimate, which can differ from your bill. Spend from subagents counts toward the cap. Spend can pass the cap, so leave headroom. When you return to a conversation with `--continue` or `--resume`, totals restored from earlier runs don't count toward it. Once spend reaches the cap, spawning another subagent fails with `Budget limit reached`, and Claude Code stops background subagents that are still running" (v2.1.217+).

- Headroom: the check runs after each model response, so "spend can therefore pass the cap by up to the cost of that one response, plus anything that subagents still running at that moment spend before they stop."
- Result: `subtype: "error_max_budget_usd"`, no `result` field, non-zero exit; `usage` omits the response that crossed the cap but `total_cost_usd`/`modelUsage` include it.
- If the cap is hit while waiting for background work at exit, "Claude Code stops the remaining background work instead of waiting."
- SDK equivalents: `maxBudgetUsd` (TS) / `max_budget_usd` (Python).

### `--max-turns <n>` (turn cap)

"Limit the number of agentic turns (print mode only). Exits with an error when the limit is reached. No limit by default." Counts tool-use turns. Result `subtype: "error_max_turns"`. SDK: `maxTurns` / `max_turns`.

### Subagent caps (env vars)

`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` (default 3; `1` turns nesting off) and `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` (default 20), both v2.1.217+.

### Task budgets (Messages API beta) - not a CLI feature

The Claude API has a separate "task budget": `output_config.task_budget: {"type": "tokens", "total": <int>, "remaining"?: <int>}` with beta header `task-budgets-2026-03-13`. It is "an advisory token budget for the full agentic loop" that the model sees as a server-injected countdown; "Task budgets are a **soft hint, not a hard cap**"; minimum `total` is 20,000; `max_tokens` remains the enforced per-request limit. Supported on Fable 5/5.1, Opus 5/5.5, Opus 4.7/4.8, Sonnet 5.5, Haiku 5.5 (not Sonnet 5, Opus 4.6, Sonnet 4.6, Haiku 4.5).

The same page states: "Task budgets are not supported on Claude Code or Cowork surfaces. Use task budgets directly through the Messages API on a supported model."

However, the Python Agent SDK reference documents `ClaudeAgentOptions.task_budget: TaskBudget | None` - "API-side token budget. Sent as `output_config.task_budget` with the `task-budgets-2026-03-13` beta header. Pass `{"total": <int>}`." (`TaskBudget` is a `TypedDict` with `total: int`.) There is no corresponding `claude` CLI flag in `--help` 2.1.303 or the CLI reference. Treat the SDK option as the only documented way to attach a task budget through Claude Code tooling, and the contradiction between the two pages as unresolved (section 9).

So for the CLI, "task budget / max spend" = `--max-budget-usd` (hard, dollar-denominated, estimate-based) plus `--max-turns`; the API task budget is token-denominated and advisory.

## 7. The ready-made anti-fabrication instruction

Source: [Prompting Claude Fable 5, "Ground progress claims during long runs"](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5#ground-progress-claims-during-long-runs).

Context sentence from the page: "On long autonomous runs, instruct Claude Fable 5 to audit progress against actual tool results. In Anthropic's testing, this nearly eliminated fabricated status reports even on tasks designed to elicit them:"

Verbatim instruction:

```text
Before reporting progress, audit each claim against a tool result from this session. Only report work you can point to evidence for; if something is not yet verified, say so explicitly. Report outcomes faithfully: if tests fail, say so with the output; if a step was skipped, say that; when something is done and verified, state it plainly without hedging.
```

For the CLI, this belongs in `--append-system-prompt` (or `--append-system-prompt-file`), which "preserves the default tool guidance, safety instructions, and coding conventions". The Fable 5.1 guide says "Your existing Claude Fable 5 prompts should perform well on Claude Fable 5.1 without changes" and does not restate this block.

Related first-party instructions for long autonomous runs (same page family):

- Fable 5, "Rare cases of early stopping", system reminder for autonomous pipelines (verbatim): "You are operating autonomously. The user is not watching in real time and cannot answer questions mid-task, so asking "Want me to…?" or "Shall I…?" will block the work. For reversible actions that follow from the original request, proceed without asking. Offering follow-ups after the task is done is fine; asking permission after already discussing with the user before doing the work is not. Before ending your turn, check your last paragraph. If it is a plan, an analysis, a question, a list of next steps, or a promise about work you have not done ("I'll…", "let me know when…"), do that work now with tool calls. End your turn only when the task is complete or you are blocked on input only the user can provide."
- Fable 5.1, ["Finish the whole task"](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-1#finish-the-whole-task): an expanded version of the block above, ending with "Before running a command that changes system state (such as restarts, deletes, or config edits), check that the evidence actually supports that specific action."
- Fable 5, "Recommended scaffolding changes": "Make self-verification explicit in long-run prompts. Separate, fresh-context verifier subagents tend to outperform self-critique." Suggested text: `Establish a method for checking your own work at an interval of [X] as you build. Run this every [X interval], verifying your work with subagents against the specification.`
- Opus 5.5, ["Unattended agentic runs"](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5#unattended-agentic-runs): harness-side guidance - "Treat a text-only end of turn as a report rather than as proof the task is done. Keep the task's parts in a checklist the model updates ... stop after two or three automatic continuations on the same task rather than repeating them indefinitely".
- Cross-model [Prompting best practices, "Minimizing hallucinations in agentic coding"](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices#minimizing-hallucinations-in-agentic-coding): the `<investigate_before_answering>` block ("Never speculate about code you have not opened ... Never make any claims about code before investigating unless you are certain of the correct answer").

## 8. Reference invocations (assembled from documented flags)

```bash
# Locked-down CI run: no hooks/plugins/MCP, explicit tools, hard caps, JSON out
ANTHROPIC_API_KEY=... claude --bare -p "Run the test suite and fix any failures" \
  --model sonnet --effort medium \
  --permission-mode dontAsk --permission-prompts none \
  --allowedTools "Read,Edit,Bash(npm test *)" \
  --max-turns 30 --max-budget-usd 2.00 \
  --append-system-prompt-file ./ground-progress.txt \
  --output-format json --no-session-persistence \
  | jq '{subtype, is_error, num_turns, total_cost_usd, modelUsage, permission_denials, result}'

# Plan only (no edits), then resume the same session to execute
sid=$(claude -p "Plan the auth refactor" --permission-mode plan --output-format json | jq -r '.session_id')
claude -p "Implement the plan" --resume "$sid" --permission-mode acceptEdits --output-format json

# Sandboxed shell with a strict network allowlist, injected via --settings
claude -p "Install deps and run lint" \
  --settings '{"sandbox":{"enabled":true,"allowUnsandboxedCommands":false,"network":{"allowedDomains":["registry.npmjs.org"],"strictAllowlist":true}}}' \
  --permission-mode acceptEdits --allowedTools "Bash" --output-format json
```

## 9. Could not verify / open questions

- **`--max-turns` absent from local `claude --help` (2.1.303)**. Documented in the CLI reference, which warns the help text is incomplete. Not exercised locally.
- **Task budgets on Claude Code**: the API page says "not supported on Claude Code or Cowork surfaces", while the Python SDK reference documents `task_budget`. No CLI flag exists either way. Unresolved; if needed, test `ClaudeAgentOptions(task_budget={"total": 50000})` against a supported model.
- **Approving a plan headlessly**: plan mode's blocks hold in `-p`, but the docs describe plan approval only as an interactive prompt. The section-8 pattern (plan in one `-p` run, `--resume` with a different mode) is an inference from documented pieces, not a documented recipe.
- **Exact JSON shape of the CLI's `--output-format json` object** is taken from the SDK's `SDKResultMessage` type (the headless page says the result message is the same) rather than from a live run; no API call was made for this research to avoid spend. A one-line `claude --bare -p "ok" --output-format json --model haiku --max-budget-usd 0.05` would confirm field names.
- **Fable 5.1-specific wording** of the grounding instruction: the 5.1 guide does not repeat it; the verbatim text above is from the Fable 5 guide, which the 5.1 guide says still applies.
- **`--effort ultracode`** appears in the CLI reference but not in local `--help` choices.
- **Env-vars page** was only read to the 100k-character mark; no budget-related env var (e.g. a `CLAUDE_CODE_MAX_BUDGET_USD`) was found in that portion and none is referenced anywhere else, so assume none exists.
- Context7 (`ctx7`) was used for three queries; it returned excerpts of the same first-party pages, cited above by URL, and surfaced no additional facts.

## Sources

Claude Code docs (code.claude.com):

- https://code.claude.com/docs/en/headless
- https://code.claude.com/docs/en/cli-reference
- https://code.claude.com/docs/en/permission-modes
- https://code.claude.com/docs/en/permissions
- https://code.claude.com/docs/en/sandboxing
- https://code.claude.com/docs/en/model-config
- https://code.claude.com/docs/en/costs
- https://code.claude.com/docs/en/env-vars
- https://code.claude.com/docs/en/agent-sdk/cost-tracking
- https://code.claude.com/docs/en/agent-sdk/agent-loop
- https://code.claude.com/docs/en/agent-sdk/typescript
- https://code.claude.com/docs/en/agent-sdk/python
- https://code.claude.com/docs/en/agent-sdk/subagents

Claude API / prompting docs (platform.claude.com):

- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5
- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-1
- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5
- https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- https://platform.claude.com/docs/en/build-with-claude/task-budgets

Local: `claude --version`, `claude --help` (Claude Code 2.1.303, macOS).
