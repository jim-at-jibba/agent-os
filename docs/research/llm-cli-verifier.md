# Research: cross-family verifier via the `llm` CLI

Resolves issue #12. Researched 2026-10-19 against primary sources only: the
`llm` docs at llm.datasette.io, the `llm` and `llm-openrouter` source code, the
OpenRouter docs and public models API, and the locally installed `llm` binary.
No live model calls were made (see "Not verified" at the end).

Location note: this repo had no `docs/` directory before this file, so
`docs/research/` is a new convention rather than a match to an existing one.

## TL;DR

- `llm` (0.35 locally) can call any configured model from a shell script with a
  fixed system prompt (`-s`), a pinned model id (`-m`), a sampling temperature
  (`-o temperature 0`) and a JSON schema (`--schema`), and prints the model's
  JSON object to stdout. Schema + system prompt + model + options can be frozen
  into a template (`--save`) or a YAML file checked into the repo and run with
  `-t`.
- Non-Anthropic models reach `llm` either natively (OpenAI models ship in core)
  or through the `llm-openrouter` plugin, which exposes every OpenRouter model as
  `openrouter/<vendor>/<model>` and turns on schema support only for models whose
  OpenRouter listing advertises `structured_outputs`.
- Five workable verifier candidates today, all reasoning-capable with
  `structured_outputs` on OpenRouter: `openai/gpt-5.6-terra`
  ($2.00 / $12.00), `google/gemini-3.8-flash` ($0.825 / $4.125),
  `x-ai/grok-4.7` ($2.00 / $6.00), `qwen/qwen3.8-max-0902` ($2.00 / $6.00),
  `deepseek/deepseek-v4.1-flash-1017` ($0.30 / $1.20). Prices are USD per
  million input / output tokens from the OpenRouter models API on 2026-10-19.

## Local state (what is actually installed here)

Observed by running the binary on 2026-10-19:

| Check | Result |
|---|---|
| `llm --version` | `llm, version 0.35` (uv tool install at `~/.local/share/uv/tools/llm`) |
| `llm plugins` | only `llm-cmd-comp 1.2.0`; **`llm-openrouter` is not installed** |
| `llm models default` | `gpt-5.6-luna` (OpenAI, built-in) |
| `llm models list` | 38 built-in OpenAI models (`gpt-4o` ... `gpt-5.6-sol/terra/luna`, `gpt-6-astra`, `o3`, `o4-mini`); no OpenRouter entries |
| `llm keys list` | `No keys found` |
| `llm templates path` | `/Users/mrbest/Library/Application Support/io.datasette.llm/templates` |

So the only non-Anthropic family reachable today without further setup is
OpenAI, and only after `llm keys set openai`. Everything else below needs
`llm install llm-openrouter` plus `llm keys set openrouter`.

## 1. Calling a model from a shell script with a fixed system prompt and JSON out

### The flags

From `llm prompt --help` (local, 0.35) and
<https://llm.datasette.io/en/stable/usage.html>:

| Flag | Purpose |
|---|---|
| `-s, --system TEXT` | System prompt. Docs: "Different models support system prompts in different ways"; OpenAI models are "particularly good at using system prompts as instructions." |
| `-m, --model TEXT` | Model id or alias. |
| `-o, --option KEY VALUE` | Model option, e.g. `-o temperature 0`, `-o reasoning_effort low`. |
| `--schema TEXT` | "Schema DSL, JSON schema, filepath or ID". Requests one JSON object. |
| `--schema-multi TEXT` | Same, but asks for several objects returned under an `"items"` array. |
| `--no-stream` | Do not stream; the whole response is printed at once. |
| `-n, --no-log` | Do not write the prompt/response to the SQLite log (logging is on by default). |
| `-R, --hide-reasoning` | Hide reasoning output. |
| `--json` | Emit the full response envelope (same shape as `llm logs --json`) instead of just the text. |
| `-x, --extract` | Return only the first fenced code block (fallback for models without schema support). |
| `--key TEXT` | API key override; prefer `llm keys set <name>` or the provider env var so keys stay out of scripts. |

Stdin is prepended to the prompt: `cat myscript.py | llm 'explain this code'`
(usage docs). So the work to be verified can be piped in and the instruction
kept in the system prompt.

### Schemas

Source: <https://llm.datasette.io/en/stable/schemas.html> and `llm prompt --help`.

- Concise DSL: `NAME [TYPE]: DESCRIPTION`, comma- or newline-separated. Types
  are `str` (default), `int`, `float`, `bool`. **Every listed field is required**
  and the DSL has no enums or arrays. Preview the generated JSON Schema with
  `llm schemas dsl 'name, age int'`.
- Full JSON Schema is accepted inline or as a file path:
  `llm --schema dogs.schema.json 'a surprising dog'`. The docs recommend a
  top-level object (not array); for lists, wrap in an object with an `items`
  array. "Different models support different subsets of JSON Schema, so test
  against your model."
- Output is printed as JSON. Logged data can be re-read with
  `llm logs -c --data` / `--data-key items` / `--data-array`.
- Every schema used is logged; `llm schemas` lists them and a hex id can be
  reused with `--schema <id>`.
- Model support: core supports schemas for OpenAI, Anthropic and Gemini models;
  other models get it from plugins. `llm models --schemas` lists schema-capable
  models.
- Behaviour on an unsupported model (from the installed source,
  `llm/models.py` lines 1170-1171, 0.35): `llm` raises
  `ValueError("<model> does not support schemas")` before making a request. It
  does not silently fall back to prompting for JSON. The docs page does not
  state this; the source does.

### Pinning model + temperature

- Per call: `-m openrouter/x-ai/grok-4.7 -o temperature 0`.
- Persistent default model: `llm models default <model-id>` (local
  `llm models --help`: "Show or set the default model") or `export LLM_MODEL=...`
  for the current shell (usage docs).
- Persistent default option per model:
  `llm models options set <model-id> temperature 0` (usage docs; local
  `llm models options` subcommand).
- `llm -m <model> --options` prints the option names a given model accepts.
  Locally, `gpt-5.5` lists `temperature`, `max_tokens`, `top_p`, `seed`,
  `json_object`, `reasoning_effort` (low/medium/high), `reasoning_summary`,
  `verbosity`, `service_tier`, and features `streaming, schemas, tools, async`.

### Templates (freeze system prompt + model + options + schema)

Source: <https://llm.datasette.io/en/stable/templates.html> and the schemas page.

Save once:

```sh
llm -m openrouter/google/gemini-3.8-flash \
    -s "$(cat verifier-system.txt)" \
    --schema verifier.schema.json \
    -o temperature 0 \
    --save verify
```

Then in the script: `git diff main... | llm -t verify --no-stream -n`.

Template YAML keys the docs list: `prompt`, `system`, `model`, `options`
(e.g. `temperature`), `schema_object`, `defaults`, `fragments`,
`system_fragments`, `tools`, `functions`, `extract`, `name`. `llm templates show
verify` prints the stored YAML; `llm templates path` shows the directory.

A template can also be a file path or URL: `llm -t path/to/verify.yaml` --
this is the natural way to commit the verifier config into the repo instead of
relying on the user's templates directory. Templates loaded from URLs (and
plugin loaders) have `functions:` ignored for safety. Multiple `-t` flags
combine left-to-right, so a repo can keep a model/options template separate
from the prompt template. `-m` on the command line overrides the template's
`model`.

### Minimal working recipe

```sh
#!/usr/bin/env bash
# verify.sh -- independent verdict from a non-Anthropic model.
# Requires: llm, llm-openrouter, and `llm keys set openrouter` (or OPENROUTER_KEY).
set -euo pipefail

MODEL="${VERIFIER_MODEL:-openrouter/google/gemini-3.8-flash}"
SYSTEM_PROMPT='You are an independent reviewer. Judge whether the submitted work satisfies the task. Be strict and concrete.'

# JSON Schema file rather than the DSL: the DSL cannot express enums or arrays.
cat > /tmp/verifier.schema.json <<'EOF'
{
  "type": "object",
  "properties": {
    "verdict":    {"type": "string", "enum": ["pass", "fail"], "description": "Overall judgement"},
    "confidence": {"type": "number", "description": "0 to 1"},
    "issues":     {"type": "array", "items": {"type": "string"}, "description": "Concrete problems found, empty if none"}
  },
  "required": ["verdict", "confidence", "issues"],
  "additionalProperties": false
}
EOF

llm -m "$MODEL" \
    -s "$SYSTEM_PROMPT" \
    --schema /tmp/verifier.schema.json \
    -o temperature 0 \
    --no-stream -n -R \
  | jq .
```

Flag-by-flag justification is in the tables above. Pipe the artefact to verify
on stdin (`... | ./verify.sh`). Drop `-o temperature 0` for OpenAI `gpt-5.x`
ids on OpenRouter -- see the caveat in section 2.

## 2. Reaching non-Anthropic models: `llm-openrouter`

Source: README at <https://github.com/simonw/llm-openrouter> and the plugin
source `llm_openrouter.py` (main branch, `pyproject.toml` version 0.7.1,
requires `llm>=0.34`).

- Install: `llm install llm-openrouter`. Key: `llm keys set openrouter` or
  `OPENROUTER_KEY` env var. `register_models` returns early if no key is set,
  so OpenRouter models only appear in `llm models list` once the key exists
  (source, `register_models`).
- Model ids: `openrouter/<vendor>/<model>`, e.g.
  `openrouter/google/gemini-3.8-flash`. Aliases:
  `llm aliases set verifier openrouter/x-ai/grok-4.7`.
- Listing and pricing: `llm openrouter models` prints per-model details
  including "prompt $0.7/M, completion $0.7/M"; `--json` gives the raw pricing
  object; `--free` filters free models; `llm openrouter refresh` busts the
  1-hour cache. The plugin reads `https://openrouter.ai/api/v1/models`
  (source line 24), which is the same endpoint the price table below is built
  from.
- Schema support is derived per model:
  `supports_schema=has_parameter(model_definition, "structured_outputs")`
  (source, `register_models`). So `--schema` works for exactly the models whose
  OpenRouter listing includes `structured_outputs`; for any other model `llm`
  raises "does not support schemas". README example:
  `llm -m openrouter/google/gemini-2.5-flash 'invent 3 cool capybaras' --schema-multi 'name,bio'`
  with the warning that "quality varies, so test carefully".
- Transport: models are registered as `OpenRouterResponses` (OpenRouter's
  Responses API) by default; `-o chat_completions 1` forces the older Chat
  Completions path (README + source).
- Options: the plugin's `Options` class extends the core OpenAI options, so
  `temperature`, `max_tokens`, `top_p`, `seed` etc. are available, plus
  OpenRouter extras: `reasoning_effort` ("none", "minimal", "low", "medium",
  "high", "xhigh", "max"), `reasoning_summary`, `reasoning_max_tokens`,
  `reasoning_enabled`, and `provider` (a JSON object for provider routing,
  e.g. `-o provider '{"quantizations": ["fp8"]}'`).
- Provider routing for structured output: OpenRouter's docs
  (<https://openrouter.ai/docs/features/structured-outputs>) say support is
  per endpoint, can change over time, and that to route only to endpoints that
  support it you set `require_parameters: true` in provider preferences. Via the
  plugin that is `-o provider '{"require_parameters": true}'`. The docs also say
  `strict: true` is honoured natively by some providers and treated as a hint by
  others, and that a request fails outright on a model with no structured output
  support.

## 3. Candidate verifier models (non-Anthropic)

Selection criteria: reasoning-capable (`reasoning` in OpenRouter
`supported_parameters`), `structured_outputs` advertised (so `--schema` works
through `llm-openrouter`), recent, and from a vendor positioned for coding /
agentic review. Pricing is the headline `pricing.prompt` / `pricing.completion`
from `https://openrouter.ai/api/v1/models` (USD per token, multiplied by 1e6),
fetched 2026-10-19, cross-checked against
`https://openrouter.ai/api/v1/models/<id>/endpoints`. The vendor's own direct
API price was not checked (see "Not verified").

| Model id (prefix with `openrouter/`) | Input $/M | Output $/M | Context | `temperature` | Notes | Source |
|---|---|---|---|---|---|---|
| `openai/gpt-5.6-terra` | 2.00 | 12.00 | 1,050,000 | **no** | OpenAI's mid tier "for everyday coding, reasoning, and agentic" work. Also callable natively as `-m gpt-5.6-terra` with an OpenAI key (built into `llm` 0.35). `reasoning_effort` + `verbosity` supported. Budget sibling `openai/gpt-5.6-luna` is $0.20 / $1.20 with the same parameter set. | <https://openrouter.ai/openai/gpt-5.6-terra> |
| `google/gemini-3.8-flash` | 0.825 | 4.125 | 1,048,576 | yes (Google AI Studio endpoints only) | "Google's most intelligent Flash model" with gains in "software engineering, agentic tasks, and multi-step reasoning". Reasoning tokens billed at the output rate (`internal_reasoning` = 4.125). Cheaper endpoints at $0.375 / $1.875 and $0.75 / $3.75 exist; `temperature` is only accepted on the Google AI Studio endpoints, not the Vertex ("Google") ones. | <https://openrouter.ai/google/gemini-3.8-flash> |
| `x-ai/grok-4.7` | 2.00 | 6.00 | 500,000 | yes | Listed as "SpaceXAI: Grok 4.7"; description: "particularly strong at long-running software engineering tasks, verifying its own work". All five xAI endpoints advertise `structured_outputs` and `temperature`. Supports `logprobs`. | <https://openrouter.ai/x-ai/grok-4.7> |
| `qwen/qwen3.8-max-0902` | 2.00 | 6.00 | 1,000,000 | yes | 2.4T-parameter MoE, Alibaba-only endpoints (two, both with `structured_outputs`). Dated snapshot, so the id is stable. | <https://openrouter.ai/qwen/qwen3.8-max-0902> |
| `deepseek/deepseek-v4.1-flash-1017` | 0.30 | 1.20 | 1,048,576 | yes | Two-day-old dated snapshot (17 Oct 2026); cheapest serious reasoning option. Caveat: DeepSeek's own endpoint and Alibaba's do **not** advertise `structured_outputs`; third-party endpoints (Fireworks, Together, DeepInfra, Parasail...) do, some at $0.05-0.15 / $0.15-0.6. Use `-o provider '{"require_parameters": true}'` so `--schema` is not routed to an endpoint that will reject it. | <https://openrouter.ai/deepseek/deepseek-v4.1-flash-1017> |

Honourable mention: `z-ai/glm-5.3` ($0.079 / $1.40 headline, 1M context,
reasoning + `structured_outputs`) is the cheapest of all, but it has ~45
endpoints with wildly different prices ($0.04 to $2.80 input) and inconsistent
`structured_outputs` support, so pinning a provider would be mandatory.

Why `temperature` is "no" for the OpenAI ids: OpenRouter's listing for
`openai/gpt-5.6-*` omits `temperature` from `supported_parameters` (all seven
endpoints). OpenAI reasoning models control variability through
`reasoning_effort` instead. What OpenRouter does if you send it anyway was not
tested (see below).

## 4. Practical recommendations for agent-os

1. Commit a `verify.yaml` template (keys: `system`, `schema_object`, `options:
   {temperature: 0}`) and run it with `llm -t ./verify.yaml`, overriding `-m`
   from an env var. That keeps the system prompt and schema versioned with the
   repo rather than in the user's templates directory.
2. Use a JSON Schema file, not the DSL: the DSL cannot express `enum` for a
   `pass`/`fail` verdict or arrays of issues.
3. Default to `openrouter/google/gemini-3.8-flash` for cost/quality, with
   `openrouter/x-ai/grok-4.7` or `openrouter/openai/gpt-5.6-terra` as the
   "second family" fallback. Keep all three in a different family from the
   agent doing the work.
4. Always pass `--no-stream -n -R` in scripts: one JSON blob on stdout, no
   SQLite logging of possibly-sensitive diffs, no reasoning text mixed in.
5. For OpenRouter models, add `-o provider '{"require_parameters": true}'`
   whenever `--schema` is used, because support is per endpoint.

## Not verified / flags

- **No live call was made.** No API keys are configured on this machine and
  `llm-openrouter` is not installed, so the recipe above is assembled from docs
  and source, not executed. The first real run should confirm that the JSON
  lands on stdout cleanly with `--no-stream -R`.
- **Vendor-direct pricing** (platform.openai.com, ai.google.dev, x.ai, etc.)
  was not fetched; all prices are OpenRouter's listed prices, which may include
  a routing margin and differ per endpoint. The OpenRouter HTML model pages
  rendered as `llms.txt` without the headline price, so the API JSON is the
  source of record here.
- **`temperature` on OpenAI ids via OpenRouter**: whether OpenRouter rejects or
  silently drops the parameter was not confirmed.
- **JSON reliability / instruction following** was not benchmarked. The
  `structured_outputs` flag is a capability proxy; the `llm-openrouter` README
  itself warns that quality varies across models. The candidates were chosen on
  vendor positioning and parameter support, not measured accuracy.
- Model names like "SpaceXAI: Grok 4.7" and the `gpt-5.6-sol/terra/luna` tiers
  are reproduced as OpenRouter and `llm` report them; the naming was not
  cross-checked against vendor announcements.
- The `llm` usage page reached today says the default model is `gpt-5.6-luna`;
  `llm models default` locally agrees. Older docs/README text may still say
  `gpt-4o-mini`.

## Sources

- `llm` usage docs: <https://llm.datasette.io/en/stable/usage.html>
- `llm` schemas docs: <https://llm.datasette.io/en/stable/schemas.html>
- `llm` templates docs: <https://llm.datasette.io/en/stable/templates.html>
- `llm` CLI reference (via Context7, `/websites/llm_datasette_io`): `llm models default`, `llm models --schemas`
- `llm` installed source 0.35: `llm/models.py` (schema support check), `llm/cli.py` (`--schemas` flag)
- `llm-openrouter` README and source: <https://github.com/simonw/llm-openrouter> (`llm_openrouter.py`, `pyproject.toml` 0.7.1)
- OpenRouter structured outputs: <https://openrouter.ai/docs/features/structured-outputs>
- OpenRouter models API: <https://openrouter.ai/api/v1/models> and <https://openrouter.ai/api/v1/models/{id}/endpoints> (fetched 2026-10-19)
- Local: `llm --help`, `llm prompt --help`, `llm models --help`, `llm models list --options -q gpt-5.5`, `llm plugins`, `llm keys list`, `llm templates path`
