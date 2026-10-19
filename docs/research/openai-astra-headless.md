# Research: running OpenAI "Astra" headless as a planning model

Ticket: issue #13. Researched 2026-10-19 against OpenAI primary sources only
(developers.openai.com API docs, the Codex docs, the `openai/codex` GitHub repo, and
`codex --help` from the official `@openai/codex` npm package, version `codex-cli 0.163.0`).
No API keys appear in this document.

## TL;DR

- "Astra" is official. The model is **GPT-6 Astra**, API id **`gpt-6-astra`**, released
  2026-09-03 and described by OpenAI as "our most capable model". It is the current OpenAI
  frontier reasoning model; `gpt-6.1-sol` (2026-09-29) is the cheaper "near-Astra" sibling.
- Headless via Codex CLI: `codex exec -m gpt-6-astra -c model_reasoning_effort="high" --json --sandbox read-only "..."`.
  stdout carries JSONL (or just the final message without `--json`); `turn.completed` events
  carry per-turn token usage (input, cached input, cache-write, output, reasoning).
- Headless via API: `POST /v1/responses` with `"model":"gpt-6-astra"`, `"reasoning":{"effort":...}`,
  `max_output_tokens`, `max_tool_calls`; `usage` returns cached/reasoning breakdowns.
- Permission modes: `--sandbox read-only | workspace-write | danger-full-access`;
  `--ask-for-approval on-request | never` (`codex exec` defaults to read-only + never).
  `--full-auto` is gone.
- Spend control: hard monthly spend limits exist at org/project level in the Platform
  dashboard (429 on breach). Codex has an experimental per-session token cap
  (`features.rollout_budget`). No dollar figure is reported by Codex; compute cost from usage.
- Caching is automatic on GPT-5.6+ (min 1,024-token prefix, 30 min TTL). Astra: $10 input /
  $1 cached / $12.50 cache write / $50 output per 1M tokens (<=272K prompts).

---

## 1. Is "Astra" an official name, and what is the model id?

Yes. **GPT-6 Astra** is an official OpenAI model; the API id is **`gpt-6-astra`**.

| Fact | Value | Source |
|---|---|---|
| Official name / id | GPT-6 Astra, `gpt-6-astra` (the only id listed; no dated snapshot shown) | https://developers.openai.com/api/docs/models/gpt-6-astra |
| Tagline | "Our most capable model for the most demanding work." | same |
| Release | 2026-09-03: "Released GPT-6 Astra, our most capable model, built for the hardest end-to-end work." | https://developers.openai.com/api/docs/changelog |
| Context window | 1,050,000 tokens (max input 922,000; max output 128,000) | model page |
| Knowledge cutoff | 2026-04-30 | model page |
| Endpoints | Responses, Chat Completions, Batch. Not Realtime/Assistants/etc. | model page |
| Reasoning effort levels | `low`, `medium`, `high`, `xhigh`, `max`; `none` returns HTTP 400 | model page; https://developers.openai.com/api/docs/guides/reasoning |
| Modalities | text + image in; text out | model page |
| Rate limits (Standard, Build tier) | 5,000 RPM / 1,000,000 TPM | model page |

Where it sits in the current lineup (changelog, same URL):

- 2026-09-03 `gpt-6-astra` (flagship).
- 2026-09-22 `gpt-6-sol`, `gpt-6-luna` (cost-efficiency tier).
- 2026-09-29 `gpt-6.1-sol`: "for complex coding and professional work at a lower cost than
  GPT-6 Astra"; its model page says "Near-Astra performance for complex work at a lower cost"
  (https://developers.openai.com/api/docs/models/gpt-6.1-sol).

So for an unattended planning loop the frontier choice is `gpt-6-astra`; `gpt-6.1-sol` is the
5x-cheaper fallback with the same 1.05M context and the same effort ladder.

**Could not verify:** the openai.com launch post (https://openai.com/index/gpt-6-astra/) and
the help.openai.com Model Release Notes both returned HTTP 403 to automated fetches, so the
release facts above come from the developer changelog and model page rather than the blog.

## 2. Pricing (per 1M tokens, USD)

Source: https://developers.openai.com/api/docs/pricing (confirmed on the model page).

| Model | Input | Cached input | Cache write | Output |
|---|---|---|---|---|
| `gpt-6-astra` | $10.00 | $1.00 | $12.50 | $50.00 |
| `gpt-6.1-sol` | $2.00 | $0.10 | $2.50 | $10.00 |
| `gpt-6-sol` | $2.00 | $0.20 | $2.50 | $10.00 |
| `gpt-6-luna` | $0.10 | $0.01 | $0.125 | $0.50 |

Modifiers (model page + pricing page):

- Prompts over 272K input tokens: 2x input and cache rates, 1.5x output, applied to the whole
  request (Astra long-context: $20 / $2 / $25 / $75).
- Batch and Flex: 50% of Standard (Astra: $5 / $0.50 / $25).
- Fast mode (renamed from Priority on 2026-07-30): 2x Standard. Ultrafast (Astra, added
  2026-09-29): $60 / $6 / $300.
- Regional processing endpoints: +10%.

Reasoning tokens are billed as output tokens (reasoning guide, section 5 below).

## 3. Headless via the Codex CLI (`codex exec`)

### 3.1 Docs location caveat

The Codex docs are linked from the repo README as `https://developers.openai.com/codex`, but
every `developers.openai.com/codex/*` page currently answers with a 308 redirect to
`https://learn.chatgpt.com/docs/*`. Both hosts are OpenAI-controlled; URLs below are given as
the canonical `developers.openai.com/codex/...` form with the redirect target in brackets.

### 3.2 Install and authenticate

- Install: `npm install -g @openai/codex` or `brew install --cask codex`
  (https://github.com/openai/codex/blob/main/README.md).
- API-key login for scripts: `printenv OPENAI_API_KEY | codex login --with-api-key`
  (https://developers.openai.com/codex/auth [learn.chatgpt.com/docs/auth]; `codex login --help`).
- One-shot key without a login file: `CODEX_API_KEY=<key> codex exec --json "..."` is
  documented for `codex exec`, `codex review`, the TypeScript SDK and `codex exec-server --remote`
  (https://developers.openai.com/codex/noninteractive [learn.chatgpt.com/docs/non-interactive-mode]).
- Billing: "OpenAI bills API key usage through your OpenAI Platform account at standard API
  rates" (auth page). API keys are "the recommended default for automation" (same page).
- Credentials file: `~/.codex/auth.json` under `CODEX_HOME`; `cli_auth_credentials_store` can be
  `file | keyring | auto | ephemeral` (auth page).

### 3.3 Core invocation

From `codex exec --help` (codex-cli 0.163.0) and the non-interactive docs page:

```sh
codex exec \
  -m gpt-6-astra \
  -c model_reasoning_effort="high" \
  --sandbox read-only \
  --json \
  -o /tmp/plan.md \
  --ephemeral \
  --skip-git-repo-check \
  "Plan today's work: ..."
```

| Flag | Meaning | Source |
|---|---|---|
| `[PROMPT]` / `-` | Prompt argument; `-` or no argument reads stdin. If both stdin and prompt are given, stdin is appended as a `<stdin>` block | `codex exec --help` |
| `-m, --model <MODEL>` | Model id | `--help`; models page shows `codex exec -m gpt-6.1-sol "..."` |
| `-c, --config <key=value>` | Override any config.toml key (dotted path, TOML value) | `--help` |
| `-p, --profile <name>` | Layer `$CODEX_HOME/<name>.config.toml` | `--help` |
| `-s, --sandbox <MODE>` | `read-only`, `workspace-write`, `danger-full-access` | `--help` |
| `--json` | "Print events to stdout as JSONL" | `--help` |
| `-o, --output-last-message <FILE>` | Write the final agent message to a file (still printed to stdout) | `--help`; noninteractive page |
| `--output-schema <FILE>` | JSON Schema the final response must conform to | `--help` |
| `--color <always\|never\|auto>` | ANSI colour in output | `--help` |
| `--ephemeral` | Do not persist session files | `--help` |
| `--skip-git-repo-check` | Allow running outside a git repo (exec otherwise requires one) | `--help` |
| `--ignore-user-config` | Skip `$CODEX_HOME/config.toml` (auth still read from `CODEX_HOME`) | `--help` |
| `--ignore-rules` | Skip execpolicy `.rules` files | `--help` |
| `-C, --cd <DIR>` | Working root | `--help` |
| `--add-dir <DIR>` | Extra writable directories | `--help` |
| `--worktree` | Run in a new managed git worktree | `--help` |
| `--approve-for-me` | Route approvals through auto-review with workspace-write sandbox | `--help` |
| `--dangerously-bypass-approvals-and-sandbox` (`--yolo`) | No sandbox, no approvals. "EXTREMELY DANGEROUS" | `--help` |
| `--enable/--disable <FEATURE>` | Same as `-c features.<name>=true/false` | `--help` |
| `codex exec resume --last [PROMPT]` / `resume <SESSION_ID>` | Continue a prior headless session | `--help`; noninteractive page |

Removed: the noninteractive page states "`--full-auto` is no longer supported"; use
`--sandbox workspace-write`.

Note: `-a/--ask-for-approval` is a top-level `codex` flag (`on-request | never`) and is not in
the `exec --help` list; `codex exec` sets the policy itself (see 4.2). Use
`-c approval_policy="never"` if you need it explicit.

### 3.4 Output formats

- Default: progress streams to stderr; only the final agent message goes to stdout. The
  human-output processor also prints a dimmed `tokens used` line with the blended total on
  stderr at the end (source: `codex-rs/exec/src/event_processor_with_human_output.rs`).
- `--json`: stdout becomes a JSON Lines stream. Event types: `thread.started`, `turn.started`,
  `turn.completed`, `turn.failed`, `item.started`/`item.completed` (`item.*`), `error`. Item types
  include agent messages, reasoning, command executions, file changes, MCP tool calls, web
  searches and plan updates (noninteractive page).
- `--output-schema schema.json -o out.json`: structured final answer written to a file.

Documented sample (noninteractive page):

```json
{"type":"thread.started","thread_id":"0199a213-81c0-7800-8aa1-bbab2a035a53"}
{"type":"turn.started"}
{"type":"item.completed","item":{"id":"item_3","type":"agent_message","text":"..."}}
{"type":"turn.completed","usage":{"input_tokens":24763,"cached_input_tokens":24448,"output_tokens":122,"reasoning_output_tokens":0}}
```

### 3.5 Model and reasoning-effort selection in Codex

- Model: `-m gpt-6-astra` or `model = "gpt-6-astra"` in `config.toml`. The Codex models page
  lists `gpt-6-astra`, `gpt-6.1-sol`, `gpt-6-luna` with `codex -m ...` examples
  (https://developers.openai.com/codex/models [learn.chatgpt.com/docs/models]).
- Effort: config key `model_reasoning_effort` with values "such as `low`, `medium`, `high`,
  `xhigh`, `max`, or `ultra`" ("Available levels depend on the model and client"); pass
  `-c model_reasoning_effort="xhigh"`. There is no dedicated CLI flag
  (https://developers.openai.com/codex/config-reference [learn.chatgpt.com/docs/config-file/config-reference]).
- Related keys: `model_reasoning_summary` (`auto|concise|detailed|none`), `model_verbosity`
  (`low|medium|high`), `service_tier` (`fast` maps to request value `priority`),
  `hide_agent_reasoning` (suppresses reasoning events in `codex exec` output).

**Could not verify:** whether `gpt-6-astra` is served to API-key Codex sessions in every
org. The models page says availability "depends on the rollout, your sign-in method, and your
client". Astra lists `low..max` on its API model page; `ultra` is mentioned only in Codex config
docs ("Max and Ultra depend on your settings"; Ultra "uses subagents").

### 3.6 Per-run token reporting in Codex

- `turn.completed.usage` in `--json` mode. The Rust/TS definitions have five fields:
  `input_tokens`, `cached_input_tokens`, `cache_write_input_tokens`, `output_tokens`,
  `reasoning_output_tokens` (`codex-rs/exec/src/exec_events.rs`; `sdk/typescript/src/events.ts`).
  The docs sample shows four (no `cache_write_input_tokens`), so treat that field as present but
  recently added.
- Codex does not print a dollar amount. Cost = usage x the pricing table in section 2.

### 3.7 Codex SDK alternative

`@openai/codex-sdk` (Node 18+) and `openai-codex` (Python 3.10+) wrap the same binary
(https://developers.openai.com/codex/sdk [learn.chatgpt.com/docs/codex-sdk]). TypeScript
`ThreadOptions` (from `sdk/typescript/src/threadOptions.ts`): `model`, `sandboxMode`,
`workingDirectory`, `skipGitRepoCheck`, `modelReasoningEffort`
(`minimal|low|medium|high|xhigh|max|ultra|persistent`), `networkAccessEnabled`,
`webSearchMode` (`disabled|cached|live`), `approvalPolicy`, `additionalDirectories`.
`CodexOptions`: `apiKey`, `baseUrl`, `config`, `configOverrides`, `env`, `codexPathOverride`.
`thread.run(prompt, { outputSchema })` returns `finalResponse`, `items`, `usage`.
Note the SDK's `ApprovalMode` type still lists `on-failure` and `untrusted`, which the docs say
are deprecated/retired (see 4.2).

## 4. Permission and sandbox modes, tool restriction

### 4.1 Sandbox modes (`--sandbox` / `sandbox_mode`)

Source: https://developers.openai.com/codex/sandboxing [learn.chatgpt.com/docs/sandboxing] and
https://developers.openai.com/codex/agent-approvals-security [learn.chatgpt.com/docs/agent-approvals-security].

| Mode | Behaviour |
|---|---|
| `read-only` | Inspect files; cannot edit or run mutating commands without approval. Default for `codex exec`. |
| `workspace-write` | Read anywhere, write inside the workspace (+ temp dirs), run routine commands. Protected paths stay read-only even inside writable roots: `.git`, `.agents`, `.codex`, `.aws`. |
| `danger-full-access` | No filesystem or network sandbox. "Only use in an isolated runner or container." |

Network: off by default inside the sandbox. Enable with `[sandbox_workspace_write]
network_access = true` or `-c 'sandbox_workspace_write.network_access=true'`. Other keys:
`sandbox_workspace_write.writable_roots`, `.exclude_tmpdir_env_var`, `.exclude_slash_tmp`.

Platform enforcement: macOS Seatbelt; Linux/WSL2 `bubblewrap` (`bwrap`) with a bundled
fallback needing unprivileged user namespaces; Windows native sandbox or WSL2 (sandboxing page).

Newer permission profiles (https://developers.openai.com/codex/permissions
[learn.chatgpt.com/docs/permissions]): `default_permissions = ":read-only" | ":workspace" |
":danger-full-access"` or a custom `[permissions.<name>]` table with `filesystem` rules
(`read|write|deny`) and `[permissions.<name>.network]` (`enabled`, `domains`, `proxy_url`, ...).
Not combinable with `sandbox_mode`/`--sandbox` in the same config.

### 4.2 Approval policy (`-a` / `approval_policy`)

- Values: `on-request` ("The model decides when to ask") and `never` ("Never ask for user
  approval. Execution failures are immediately returned to the model") (`codex --help`).
  A `{ granular = { ... } }` form exists; unlisted categories are auto-rejected.
- `untrusted` is retired and `on-failure` deprecated (config reference).
- `codex exec` hard-codes `approval_policy = Never` for headless runs unless the resolved
  reviewer is `auto_review` (source: `codex-rs/exec/src/lib.rs`, comment "Default to never ask
  for approvals in headless mode"). Combined with the read-only default this means a blocked
  write/command fails back to the model rather than hanging on a prompt.
- Auto-review: `approvals_reviewer = "auto_review"` (or `--approve-for-me`) sends eligible
  approval requests to a reviewer agent; it "fails closed" on errors (agent-approvals page).
- Documented presets: CI read-only is `--sandbox read-only --ask-for-approval never`;
  non-interactive edits are `codex exec --sandbox workspace-write` (agent-approvals page).

### 4.3 Restricting tools

- Web search: top-level `web_search = "disabled" | "cached" | "indexed" | "live"` (default
  `cached`; `--search` sets `live`). `tools.web_search.allowed_domains` filters results only.
- MCP: `mcp_servers.<id>.enabled`, `.enabled_tools` (allow list), `.disabled_tools` (deny list,
  applied after), `.default_tools_approval_mode = auto|prompt|writes|approve`, `.required`.
- Shell environment: `shell_environment_policy.inherit = all|core|none`, `.filters`, `.set`;
  by default variables containing KEY/SECRET/TOKEN are stripped (`ignore_default_excludes`).
- Command rules: execpolicy `.rules` files can allow/prompt/forbid command prefixes;
  `--ignore-rules` skips them. Admin lockdown via `requirements.toml`
  (`allowed_sandbox_modes`, `allowed_approval_policies`, `allowed_permission_profiles`).
- Enterprise: `forced_login_method = "api"`; `allow_managed_hooks_only`.
  (All from the config reference / permissions pages above.)

## 5. Headless via the API directly

Source: https://developers.openai.com/api/docs/guides/reasoning and the Responses reference
https://developers.openai.com/api/docs/api-reference/responses/create.

```sh
curl https://api.openai.com/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-6-astra",
    "reasoning": {"effort": "high", "summary": "auto"},
    "max_output_tokens": 40000,
    "store": false,
    "input": "Plan today..."
  }'
```

- `reasoning.effort` (Chat Completions: `reasoning_effort`): `none|minimal|low|medium|high|xhigh|max`;
  GPT-6 models default to `medium`; Astra rejects `none` with 400.
- `reasoning.summary`: `auto|concise|detailed` (raw reasoning is never returned).
- `max_output_tokens`: "An upper bound for the number of tokens that can be generated for a
  response, including visible output tokens and reasoning tokens." If hit, `status` is
  `incomplete` with `incomplete_details.reason = "max_output_tokens"`, possibly before any
  visible text; OpenAI "recommends reserving at least 25,000 tokens for reasoning and outputs".
- `max_tool_calls`: "The maximum number of total calls to built-in tools that can be processed
  in a response" (built-in tools only; it does not cap your own function calls).
- `service_tier`: `auto|default|flex|scale|priority|fast|ultrafast`.
- `store: false` (or ZDR) makes reasoning items carry `encrypted_content` you must replay
  yourself for multi-turn.
- `background: true` runs the response asynchronously.
- Usage object: `usage.input_tokens`, `usage.input_tokens_details.cached_tokens`,
  `usage.input_tokens_details.cache_write_tokens`, `usage.output_tokens`,
  `usage.output_tokens_details.reasoning_tokens`, `usage.total_tokens` (prompt-caching guide;
  reference example). No dollar amount is returned.

Other reporting: the Platform dashboard Usage/Costs pages; no per-request cost field exists.

## 6. Spend caps, task budgets, max tokens

| Mechanism | What it does | Source |
|---|---|---|
| Org/project **hard spend limit** (added 2026-07-22) | Dashboard: Organization limits (or Project settings > Limits) > Spend > Edit spend limit > Monthly spend limit + "Enforce a hard limit". Breach returns HTTP 429 with `organization_spend_limit_exceeded` / `project_spend_limit_exceeded`; resets with the next monthly cycle; enforcement lags slightly | https://developers.openai.com/api/docs/guides/spend-limits |
| Spend alerts | Notify at a threshold, traffic continues | same |
| Approved usage-tier limit | Separate monthly cap set by OpenAI (`organization_usage_limit_exceeded`) | same |
| API `max_output_tokens` | Per-response cap incl. reasoning | section 5 |
| API `max_tool_calls` | Per-response cap on built-in tool calls | section 5 |
| Codex `features.rollout_budget` | `enabled`, `limit_tokens` (required), `reminder_interval_tokens`, `sampling_token_weight`, `prefill_token_weight`. "This feature is under development and off by default." Source shows that when the weighted total (output tokens x sampling weight + non-cached input x prefill weight) reaches `limit_tokens` the session raises `SessionBudgetExceeded` and the turn aborts with `TurnAbortReason::BudgetLimited` | config reference; `codex-rs/core/src/rollout_budget.rs`, `codex-rs/core/src/agent/control/budget.rs`, `codex-rs/core/src/tasks/mod.rs` |
| Codex `model_auto_compact_token_limit`, `tool_output_token_limit`, `model_context_window` | Context-size controls, not spend caps | config reference |

**Could not verify / absent:** no `max_turns`, wall-clock, or dollar budget key exists in Codex;
no `model_max_output_tokens` key appears in the current config reference; the spend-limits page
does not say explicitly that Codex API-key traffic is covered, but the auth page says such
usage is billed through the Platform account at API rates, so project limits should apply to
the project the key belongs to (inference, not a documented statement). Use a dedicated
project + key for the daily loop so the project hard limit scopes it.

## 7. Prompt caching

Source: https://developers.openai.com/api/docs/guides/prompt-caching plus the Responses
reference and pricing page.

- "Prompt caching is enabled by default for supported OpenAI models." Hits require the full
  rendered prefix to match; put stable content (instructions, tools, reference docs) first and
  dynamic content (dates, today's task) last.
- GPT-5.6 and later (incl. GPT-6 Astra): minimum cacheable prefix 1,024 tokens; TTL set via
  `prompt_cache_options.ttl`, currently only `"30m"` (default) - a prefix stays eligible for
  at least 30 minutes after its last write or reuse. `prompt_cache_options.mode = "explicit"`
  plus `prompt_cache_breakpoint` items allow up to four explicit breakpoints per request.
  `prompt_cache_retention` (`in_memory|24h`) is deprecated for these models.
- `prompt_cache_key` is optional on GPT-5.6+ (mainly separates accounting per user/customer).
- Pricing for GPT-5.6+/GPT-6: cache reads 0.1x input (0.05x for `gpt-6.1-sol`), cache writes
  1.25x input. For Astra: $1 read / $12.50 write vs $10 uncached. Cached tokens still count
  toward TPM rate limits; caches cannot be cleared manually.
- Reporting: `usage.input_tokens_details.cached_tokens` and `.cache_write_tokens`.
- In Codex: the CLI sets `prompt_cache_key` automatically (session id, or
  `<source>:<parent_thread_id>` for subagents; `codex-rs/core/src/client.rs`), and surfaces
  `cached_input_tokens` / `cache_write_input_tokens` in `turn.completed.usage`. For a daily
  loop that starts a fresh session each day with >30 min gaps, expect a cache write (1.25x)
  on the first call and reads only within the same run.

## 8. Suggested baseline for the daily loop (derived from the above)

```sh
export CODEX_API_KEY="$(cat /path/to/key)"   # or: printenv OPENAI_API_KEY | codex login --with-api-key
codex exec \
  -m gpt-6-astra \
  -c model_reasoning_effort="high" \
  -c 'web_search="disabled"' \
  --sandbox read-only \
  --ephemeral --skip-git-repo-check \
  --json -o "$OUT/plan.md" \
  --output-schema "$SCHEMA" \
  - < "$PROMPT_FILE" > "$OUT/events.jsonl"
jq -r 'select(.type=="turn.completed") | .usage' "$OUT/events.jsonl"
```

Guard with a project-level hard spend limit in the Platform dashboard, and optionally
`--enable rollout_budget -c features.rollout_budget.limit_tokens=200000` (experimental).

## 9. Unverified items, in one place

1. openai.com launch post and help.openai.com release notes: HTTP 403 to automated fetch;
   release date taken from the developer changelog.
2. Codex docs host: `developers.openai.com/codex/*` currently 308-redirects to
   `learn.chatgpt.com/docs/*`; treated as the same official docs.
3. Astra availability to API-key Codex sessions is "rollout"-dependent per the models page.
4. `ultra` effort is documented for Codex only; Astra's API page lists `low..max`.
5. `features.rollout_budget` is marked "under development"; exhaustion behaviour read from
   source, not docs.
6. Whether Platform spend limits cover Codex-via-API-key traffic is inferred from billing
   statements, not stated on the spend-limits page.
7. `cache_write_input_tokens` in Codex JSONL usage is in source and SDK types but not in the
   documented sample.
