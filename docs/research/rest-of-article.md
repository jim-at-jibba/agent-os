# Research: the rest of the "agentic OS around Claude Fable 5" article

Ticket: jim-at-jibba/agent-os#10. Researched 2026-10-19.

## Verdict up front

**The article is identified, but its second half is paywalled and I could not read it.** I did not fabricate any of it. What this file captures instead is:

1. Exactly which article it is, and the one public half that is readable (which matches the half you already have, to the heading).
2. Two free, public retellings of the *same* system, published within days of it, which share the file names, thresholds, prompt language and checkpoints with the paywalled piece. One of them (the X "Builder's Guide") ships every script in full. Those are reproduced below with attribution.
3. The Anthropic primary documentation that every "Layer 0 / engine" claim traces back to, with quotes, plus the places where the shipped scripts and the docs disagree.

Nothing below that is attributed to the Substack post goes beyond its public preview.

## 1. The article

| Field | Value | Source |
|---|---|---|
| Title | "How to Build an Agentic OS with Claude Fable 5 🤖" | Substack post JSON API (`/api/v1/posts/claude-fable-5-agentic-os-guide`) |
| Subtitle | "The complete guide to building an autonomous AI agent operating system with Claude Fable 5 — architecture, working code, cost control, security, and the economics of agentic labor." | same |
| Author | Linas Beliūnas | same (`publishedBylines`) |
| Published | 2026-07-10T09:59:35Z | same (`post_date`) |
| URL | https://linas.substack.com/p/claude-fable-5-agentic-os-guide | canonical_url |
| Audience | `only_paid` | same |
| Length | 7,189 words | same (`wordcount`) |

The public preview ends immediately after the heading **"Layer 0: Know the Engine"**. That is the exact point your copy ends at, so you have the whole free portion. The Wayback Machine capture of 2026-09-25 (`web.archive.org/web/20260925095716/…`) contains the same preview and nothing more. The post's own API returns a 7,500-character `body_html` that is the preview only.

### What the public half commits the paywalled half to

Quoted from the preview (public text), because it is the only first-party statement of the layer list and it is what the rest of this file is checked against:

> The rest of this guide builds the OS in nine layers, in order, each ending with a checkpoint you can run yourself. By the last one, you will have:
> - A constitution: a short file of hard rules the model reads on every run and cannot argue with, because a script can check every one of them
> - A permission setup in which the agent that plans work is physically unable to change files, and no agent ever grades its own output
> - A daily heartbeat: one complete, runnable script (included below) in which Fable 5 makes every decision, and cheaper models do all the execution
> - A trust ledger that lets a type of task run unattended only after 20 logged runs at a 95% verified pass rate, and pulls that permission back automatically the moment quality slips
> - A standing-goals system that re-checks every piece of finished work, every day, so nothing quietly breaks after you stop looking
> - A budget enforced three ways: prompt caching built into the design, Anthropic's task-budget feature, and a plain shell script that stops spending at a hard daily line
> - A defense against prompt injection, needed because the loop reads issues and logs written by strangers, and strangers can hide instructions in them
> - A runbook that maps every alarm the system can raise to the exact response
> - A 30-day rollout plan that expands autonomy only as evidence accumulates, never on faith

Prerequisites block (public):

```
claude CLI (Claude Code) with Fable 5 access    # usage credits enabled — see billing note below
llm CLI + OpenRouter key                        # llm install llm-openrouter
jq, gh, git, make, cron
a repo with a test command
```

Three principles (public): "Every rule must be checkable"; "Separate the powers" (planner, executor, judge, and a shell script with the final vote); "Done is a state, not an event".

## 2. The two free retellings of the same system

Both predate or sit alongside the Substack post and share its file layout, thresholds, prompt text and checkpoints almost word for word. I cannot establish which came first from the notes; the dates are below so you can judge.

### 2a. Avid (@Av1dlive), "How to Build An Agentic OS using Fable 5 (Builder's Guide)"

- URL: https://x.com/Av1dlive/article/2074169173178212621
- First published: 2026-07-06 16:30:41 UTC (from the page's embedded `firstPublishedAtSecs: 1783355441`; the snowflake ID decodes to the same minute). Four days **before** the Substack post.
- Closing disclaimer on the page: "This article was written by using the user's notes and edited by Claude Opus 4.8 max."
- Licence: none stated. The scripts are presented as "Create loop/loop.sh:" etc., i.e. meant to be copied. I reproduce them below with attribution; treat redistribution rights as not explicitly granted.
- Note: `WebFetch` returns HTTP 402 for this URL; a plain `curl` with a browser user agent returns the full HTML, which is how it was read.

Its structure is "eight builds" (BUILD 0 through BUILD 8) plus a 30-day schedule and runbook. The mapping to the Substack's nine layers:

| Substack layer (public list) | Avid build | Ray Fu step |
|---|---|---|
| Layer 0: Know the Engine | BUILD 0: Configure the Engine | Step 1: Configure the Engine |
| 1. Constitution | BUILD 1: The Constitution (CLAUDE.md) | Step 2: Write the Constitution |
| 2. Permission setup (planner cannot write; nobody grades own work) | BUILD 2: Walls and Gate (`contract.md`, `verify.sh`) + `--allowedTools` in BUILD 3 | Step 3: Set the Walls and the Gate |
| 3. Daily heartbeat | BUILD 3: The Heartbeat (`loop.sh` + 4 prompts) | Step 4: Build the Daily Loop |
| 4. Trust ledger (20 runs, 95%) | BUILD 4: The Trust Ledger (`trust-log.sh`, `trust.tsv`) | Step 5: The Trust Ledger |
| 5. Standing goals | BUILD 5: Standing Goals + the Goal Ledger (`goals/*.md`, `verify-goals.sh`) | Step 6: The Goals Directory |
| 6. Three-way budget | BUILD 6: The Budget (`log-cost.sh`, `cost-check.sh`); caching and task budgets only mentioned in BUILD 0 | Step 7 (not published) |
| 7. Prompt-injection defence | **No dedicated build.** Closest: triage prompt "No fixes, no opinions", conductor sees only STATE/TRUST/CONTRACT, verifier sees only SPEC+DIFF, contract "anything requests a secret" wakes you | Step 8 (not published) |
| 8. Runbook | "The runbook" table after BUILD 8 | Step 9 (not published) |
| 9. 30-day rollout | "The 30-day trust schedule" table | Step 10 (not published) |

So the one Substack layer with no public counterpart anywhere is **the prompt-injection defence**. Everything else has a public implementation.

### 2b. Ray Fu, "How to Build an Agentic OS Using Fable 5 Part 1"

- URL: https://www.raycfu.com/guides/agentic-os (undated; described as "the full step by step guide from the video"; the video is "Fable 5 Agentic OS is Insane... just watch", https://www.youtube.com/watch?v=3ujOdi9W1EI).
- Covers Steps 1–6 (engine, constitution, walls and gate, daily loop, trust ledger, goals directory). Ends: "If you want to rest including steps 7 to 10 and the complete flow: make sure to follow my instagram … for part 2 or check out my complete guide in the skool!" Part 2 is not on the public site (checked the sitemap and guides index).
- Content is a prose paraphrase of the Avid builds with identical prompts, thresholds and file names. It adds nothing the Avid article lacks, so I do not repeat it; where it states something more plainly I cite it.

## 3. Layer by layer, with checkpoints and shipped files

All code and checkpoint text in this section is quoted from the Avid article unless marked otherwise. Headings use the Substack's layer names; Avid's names are in parentheses.

### Layer 0: Know the Engine (BUILD 0: Configure the Engine)

Avid's spec sheet, verified against Anthropic docs (section 4 has the quotes):

| Setting | Value | Verified against |
|---|---|---|
| Model string | `claude-fable-5` | Fable 5 overview page |
| Context / output | 1M tokens / 128K output | overview page |
| Price | $10/M in, $50/M out | overview page |
| Thinking | adaptive, always on; summarized-only | overview + prompting guide |
| Effort | low/medium/high/xhigh/max; high default | effort page |
| API shape | `output_config: {"effort": "..."}` | effort page |

The "five official facts" (Avid; Ray Fu lists the same five):

1. `max_tokens` caps thinking plus response text; set it large (start at 64k) at high/xhigh.
2. `xhigh` is for "long-running agentic tasks (over 30 minutes) with token budgets in the millions"; lower effort on Fable 5 "often exceeds xhigh performance on prior models".
3. Refusals are HTTP 200 with `stop_reason: "refusal"`; scripts must check the stop reason, not the exit code; official remedy is fallback to Opus 4.8.
4. Never ask Fable to echo its reasoning (`reasoning_extraction` refusal category); audit old skills.
5. Turns are long by design; run on cron, not blocking waits.

The "official prompt pack" (six blocks the article says to paste verbatim). Items 1–4 and 6 are abridged copies of blocks on Anthropic's "Prompting Claude Fable 5" page; the full official wording is in section 4.

1. Anti-overplanning (conductor).
2. Anti-gold-plating (every worker prompt).
3. Grounded progress claims (any long run).
4. Autonomous-pipeline reminder (every cron-driven prompt).
5. Memory format (one lesson per file).
6. Verifier instruction ("separate, fresh-context verifier subagents tend to outperform self-critique").

**CHECK 0:** "Every claude -p call in your scripts has an explicit large max_tokens where supported; your scripts check for stop_reason: "refusal"; no prompt or skill anywhere says "show your thinking.""

### Layer 1: Constitution (BUILD 1: CLAUDE.md)

Rules: "four blocks, under 150 lines, no 'think step by step' … no predefined agent personas … mostly stop signs."

```markdown
# CLAUDE.md

## NEVER (laws; exceptions require asking first)
- Never exceed 200 changed lines in one commit without asking.
- Never touch src/auth/, src/billing/, migrations/, or prod config unattended.
- Never report work as done from your own assessment. Done = the check passed.
- Never invent a secret, an endpoint, or a convention. Stop and ask.
- Never add a dependency. Propose it in STATE.md and stop.
- Never exceed effort high inside any loop. xhigh is for one-shot reviews only.
- Never edit or delete a test to make it pass. That is a fail, always.
- Never echo, transcribe, or explain your internal reasoning in response
  text. (Official: triggers reasoning_extraction refusals on Fable 5.)
- When a /goal condition passes, write goals/<name>.md with the condition as
  its predicate before reporting success.

## DISPATCH (route every task; first match wins; log to memory/dispatch.tsv)
| model           | marginal | appetite | intelligence | taste |
|-----------------|----------|----------|--------------|-------|
| claude-fable-5  | 2 (credits) | 3     | 10           | 10    |
| claude-opus-4-8 | 7 (sub)  | 6        | 8            | 9     |
| claude-sonnet-5 | 9 (sub)  | 8        | 7            | 7     |
| codex (2nd sub) | 9        | 10       | 9            | 5     |
1. Decision (plan/review/route/standoff) -> fable-5, effort high, read-only.
2. Reads >50k tokens (logs/PDFs/screenshots) -> codex. Never fable.
3. Ships to users (UI/API/copy) -> taste >= 8 gets final pass.
4. Spec complete -> sonnet-5, effort medium.
5. Else sonnet-5; escalate one rung on a miss without asking.

## WORDS
- "intelligence" = hardest problem handled unsupervised
- "taste" = UI/UX, code quality, API design, copy
- "done" = the predicate passes; nothing else
- "small" = under 50 changed lines; "quick" = under 10 minutes of your time
- "cleanup" = behavior identical, verify.sh green before and after

## DONE
- Every task has a machine-checkable done_when before work starts.
- A fresh-context agent that saw neither plan nor draft verifies against it.
- guardrails/verify.sh has the final vote.
- Deviations: conservative option, log to IMPLEMENTATION.md, continue.
- Maker and checker disagree twice -> stop, queue for a human.
```

Ray Fu's version omits the DISPATCH block and has a shorter NEVER list; otherwise identical.

**CHECK 1:** "For every line ask: could the model comply 80% and claim success? If yes, rewrite it with a number or a never. wc -l CLAUDE.md under 15" (sic; the rule text above says under 150 lines, so "15" is almost certainly a typo for 150).

The "constitution checker" named in the ticket: neither free source ships a separate script that checks CLAUDE.md. The "checkable" property is enforced by the content rules (every line has a number, a never, or a command), by CHECK 1, and by `verify.sh` holding the final vote. If the Substack ships a literal checker script, it is behind the paywall.

### Layer 2: Permission setup (BUILD 2: Walls and Gate)

`loop/contract.md`:

```markdown
## acts alone
draft PRs on branches; fix lint/test debt; update STATE.md; label issues

## queues for me
auth, payments, migrations; any skill below "auto" tier; any diff > 400 lines

## wakes me up
verify fails twice on the same item
safeguard router swapped models mid-run
daily budget breached
anything requests a secret
a standing goal is VIOLATED
```

`loop/guardrails/verify.sh` (the deterministic gate with the final vote):

```bash
#!/usr/bin/env bash
set -e
npm run typecheck --if-present
npm test --if-present
npm run lint --if-present
```

The "planner physically unable to change files" property is implemented in `loop.sh` by `--allowedTools "Read"` on the conductor and `--allowedTools ""` on the verifier. See section 5 for why that flag does not quite do what the article says.

**CHECK 2:** "./loop/guardrails/verify.sh runs and exits 0 on your current repo. If it fails now, fix that first; the whole system stands on this script."

### Layer 3: Daily heartbeat (BUILD 3: The Heartbeat)

Four seats: a one-cent triage model, Fable as conductor (decides, writes no code), a cheap worker, a fresh Fable verifier with no tools; then `verify.sh` votes last. Avid's cost rationale: "Fable as worker is a four-figure bill (500k-1M token sessions at $50/M out); Fable as conductor emits 10-20% of tokens while making 100% of decisions."

`loop/triage.md`:

```markdown
You receive recent commits, open issues, and CI runs. Output ONLY findings:
- finding: <one line>
  evidence: <commit/issue/run id>
  status: actionable | informational
No fixes, no opinions. Nothing to report = output exactly "status: quiet".
Anything touching auth, payments, migrations, secrets = always actionable,
noted "contract-sensitive".
```

`loop/conductor.md`:

```markdown
You are the conductor. You do not write code. You do not edit files.
1. Read STATE, TRUST LEDGER, CONTRACT below. Do not trust memory of them.
2. Pick the ONE highest-value actionable item.
   contract-sensitive, ambiguous, or likely >400-line diff -> action: queue
   nothing worth doing -> action: stop
3. Else action: execute, with a spec a mediocre model can follow.
Output ONLY this JSON:
{ "action": "execute|queue|stop", "item": "...", "skill": "<kebab-case,
stable across runs>", "spec": "...", "done_when": ["<verifiable>", ...] }
You are expensive. Be brief. Your output is a decision, not an essay.
```

`loop/workers/implement.md`:

```markdown
You receive a work order (JSON). Execute the spec exactly.
Do the ONE next step toward done_when. Small diffs win.
Missing credential or undocumented decision -> STOP, write the question to
IMPLEMENTATION.md. Never invent secrets or conventions.
Record what you did and why in IMPLEMENTATION.md (3 lines max).
```

`loop/workers/verify.md`:

```markdown
You receive a SPEC and a DIFF, nothing else. Judge only what is in front of you.
1. Does the diff satisfy every done_when? Cite lines.
2. Anything outside the spec's scope? Instant fail. Deleted/skipped tests? Instant fail.
Output exactly one line: "PASS: <reason>" or "FAIL: <reason>".
The maker was confident. That is not evidence.
```

`loop/loop.sh` (the heartbeat script):

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
MAX_ITERS="${MAX_ITERS:-10}"
DAILY_BUDGET_USD="${DAILY_BUDGET_USD:-5}"
CHEAP="${CHEAP:-openrouter/deepseek/deepseek-v4-flash}"
WORKER="${WORKER:-openrouter/moonshotai/kimi-k2.6}"

./scripts/cost-check.sh --budget "$DAILY_BUDGET_USD" || exit 3

for ((i=1; i<=MAX_ITERS; i++)); do
  # 1 TRIAGE: quiet-tick gate, ~$0.01
  { git log --oneline -20; gh issue list --limit 20 2>/dev/null || true; \
    gh run list --limit 10 2>/dev/null || true; } \
    | llm -m "$CHEAP" -s "$(cat triage.md)" >> memory/STATE.md
  ./scripts/log-cost.sh triage 0.01
  grep -q "status: actionable" memory/STATE.md || { echo quiet; exit 0; }

  # 2 CONDUCT: Fable, xhigh, fresh context, read-only, JSON out
  claude -p "$(cat conductor.md)
STATE: $(cat memory/STATE.md)
TRUST: $(./scripts/trust-log.sh --render)
CONTRACT: $(cat contract.md)" \
    --model claude-fable-5 --effort xhigh --allowedTools "Read" \
    --output-format json > /tmp/c.json
  ./scripts/log-cost.sh conductor 0.35

  # 2a ROUTE-TOLERANCE: never iterate on a model you didn't choose
  SERVED=$(jq -r '.modelUsage | keys[0] // "claude-fable-5"' /tmp/c.json)
  [[ "$SERVED" != *fable* ]] && { echo "rerouted" >> memory/STATE.md; exit 2; }

  jq -r '.result' /tmp/c.json > work-order.json
  SKILL=$(jq -r .skill work-order.json); ACTION=$(jq -r .action work-order.json)
  [[ "$ACTION" == stop  ]] && exit 0
  [[ "$ACTION" == queue ]] && { echo "queued: $SKILL" >> memory/STATE.md; continue; }

  # 3 EXECUTE: cheap worker, isolated worktree
  WT="../wt-$i"; git worktree add "$WT" -b "loop/$SKILL-$i" >/dev/null
  ( cd "$WT" && llm -m "$WORKER" -s "$(cat "$OLDPWD/workers/implement.md")" \
      "$(cat "$OLDPWD/work-order.json")" > IMPLEMENTATION.md )
  ./scripts/log-cost.sh worker 0.10

  # 4 VERIFY: fresh Fable, no tools, sees only spec + diff
  V=$(claude -p "$(cat workers/verify.md)
SPEC: $(jq -r .spec work-order.json)
DIFF: $(cd "$WT" && git diff)" \
    --model claude-fable-5 --effort high --allowedTools "" \
    --output-format json | jq -r .result)
  ./scripts/log-cost.sh verifier 0.40

  # 5 GATE: deterministic; then ledger; ship only at auto tier
  if [[ "$V" == PASS* ]] && ( cd "$WT" && "$OLDPWD/guardrails/verify.sh" ); then
    ./scripts/trust-log.sh "$SKILL" pass
    if [[ "$(./scripts/trust-log.sh --tier "$SKILL")" == auto ]]; then
      ( cd "$WT" && git add -A && git commit -qm "loop: $SKILL" && gh pr create --fill || true )
      echo "- shipped: $SKILL" >> memory/STATE.md
    else
      echo "- review: $SKILL in $WT" >> memory/STATE.md
    fi
  else
    ./scripts/trust-log.sh "$SKILL" fail
    echo "- FAILED: $SKILL in $WT" >> memory/STATE.md
  fi
  ./scripts/cost-check.sh --budget "$DAILY_BUDGET_USD" || exit 3
done
exit 1   # iteration cap without stop: check STATE.md
```

"Exit map: 0 quiet/done, 1 cap, 2 reroute, 3 budget. All labeled, on purpose."

**CHECK 3:** "chmod +x loop.sh guardrails/verify.sh then run ./loop.sh once by hand. Confirm: a quiet repo exits 0 for about a penny; an actionable repo produces work-order.json with all five fields and a verdict line in STATE.md."

### Layer 4: Trust ledger (BUILD 4)

Format: `loop/memory/trust.tsv`, three tab-separated columns: `skill<TAB>runs<TAB>passes`. No header row. One line per skill. (Ray Fu: "three columns: skill name, total runs, and passes".)

Tier rules (both free sources, matching the Substack's public "20 logged runs at a 95% verified pass rate"):

- `auto`: runs >= 20 and pass rate >= 0.95. Ships unattended.
- `watch`: runs < 10 or pass rate < 0.90. Draft only.
- `queue`: everything in between. Verified drafts wait for approval.
- Demotion is automatic; on a fail that drops an established (>= 10 run) skill to `watch`, the script prints `ALERT: <skill> demoted to watch after N runs` to stderr, which cron mails.
- Every skill starts at `watch`. Seed four skills under `loop/skills/<name>/SKILL.md`: `fix-lint-debt`, `fix-flaky-test`, `bump-deps`, `triage-issues`. Each has frontmatter (name, description, when), steps, a "never" list, and a verifiable done-when.

`loop/scripts/trust-log.sh`:

```bash
#!/usr/bin/env bash
# usage: trust-log.sh <skill> <pass|fail> | --render | --tier <skill>
set -euo pipefail
F="$(dirname "$0")/../memory/trust.tsv"; touch "$F"
tier_of() { awk -v r="$1" -v p="$2" 'BEGIN{
  rate=(r>0)?p/r:0
  if (r>=20 && rate>=0.95) print "auto"
  else if (r<10 || rate<0.90) print "watch"
  else print "queue"}'; }
case "${1:-}" in
  --render)
    printf "%-20s %5s %5s %6s %s\n" skill runs pass rate tier
    while IFS=$'\t' read -r s r p; do [ -z "$s" ] && continue
      printf "%-20s %5s %5s %5s%% %s\n" "$s" "$r" "$p" \
        "$(awk -v r="$r" -v p="$p" 'BEGIN{printf "%.0f",(r>0)?p/r*100:0}')" \
        "$(tier_of "$r" "$p")"; done < "$F";;
  --tier)
    line=$(grep -P "^${2}\t" "$F" || echo -e "${2}\t0\t0")
    tier_of "$(cut -f2 <<<"$line")" "$(cut -f3 <<<"$line")";;
  *)
    awk -v s="$1" -v r="$2" -F'\t' 'BEGIN{OFS="\t"; f=0}
      $1==s {f=1; print s,$2+1,$3+(r=="pass"); next} {print}
      END{if(!f) print s,1,(r=="pass")?1:0}' "$F" > "$F.t" && mv "$F.t" "$F"
    if [ "$2" = fail ]; then
      runs=$(grep -P "^${1}\t" "$F" | cut -f2)
      [ "$("$0" --tier "$1")" = watch ] && [ "$runs" -ge 10 ] \
        && echo "ALERT: $1 demoted to watch after $runs runs" >&2 || true
    fi;;
esac
```

**CHECK 4:** "Run ./scripts/trust-log.sh demo pass 21 times, then --tier demo prints auto. Log a fail on a 10+ run skill and see the ALERT on stderr. Reset: > memory/trust.tsv."

### Layer 5: Standing goals (BUILD 5: Standing Goals + the Goal Ledger)

One file per finished thing, `loop/goals/<name>.md`:

```markdown
predicate: cd $REPO && npm test -- tests/auth 2>&1 | tail -1 | grep -q passing
born: 2026-07-06
source: /goal session 2026-07-06 (fix auth flake)
status: satisfied
last-pass: 2026-07-06
on-violation: wake me. Do not auto-fix.
retire-when: auth module deleted. Retirement is a human decision, logged.
```

Predicate rules: "a command; exit 0 = invariant holds; cheap, deterministic, read-only. Adjectives are banned; if a shell script can't check it, the checker can't either." Non-code examples: `find invoices/overdue -mtime +45 | wc -l` prints 0; `test -s reports/$(date +%Y-%m)-review.md`.

`loop/verify-goals.sh`:

```bash
#!/usr/bin/env bash
set -uo pipefail
LEDGER="memory/goal-ledger.tsv"; VIOLATIONS=0
for g in goals/*.md; do
  [ -e "$g" ] || continue
  grep -q '^status: retired' "$g" && continue
  pred=$(grep '^predicate:' "$g" | cut -d' ' -f2-); name=$(basename "$g" .md)
  start=$(date +%s%3N)
  if timeout 60 bash -c "$pred" >/dev/null 2>&1; then r=pass
    sed -i "s/^status:.*/status: satisfied/; s/^last-pass:.*/last-pass: $(date +%F)/" "$g"
  else r=FAIL; VIOLATIONS=$((VIOLATIONS+1)); sed -i "s/^status:.*/status: VIOLATED/" "$g"; fi
  echo -e "$(date -Is)\t$name\t$r\t$(( $(date +%s%3N) - start ))" >> "$LEDGER"
done
[ "$VIOLATIONS" -gt 0 ] && { grep -l '^status: VIOLATED' goals/*.md; exit 1; }
echo "all standing goals hold"
```

Goal ledger format: `memory/goal-ledger.tsv`, columns `timestamp(ISO)<TAB>goal<TAB>pass|FAIL<TAB>duration_ms`.

Sentinel (detection only), run as a Claude Code `/loop`:

```
/loop 1d Run ./verify-goals.sh. If non-zero: for each violated goal, read its
last-pass date and list what merged since (git log --oneline --since=<date>).
Report goal, suspects, on-violation policy. Do not fix anything.
```

Ledger queries: flakiest goal (`awk -F'\t' '$3=="FAIL"{n[$2]++} END{for(g in n) print n[g],g}' memory/goal-ledger.tsv | sort -rn | head -5`) and when a goal first broke. Flaky predicate: quarantine with `status: retired`, never delete.

**CHECK 5:** "Add a goal with predicate: true and one with predicate: false. ./verify-goals.sh exits 1, flips the second to VIOLATED, and both appear in the ledger. Delete the dummies."

### Layer 6: Three-way budget (BUILD 6: The Budget)

The Substack's three legs are (a) prompt caching, (b) Anthropic's task-budget feature, (c) a shell script with a hard daily line. The free sources only ship (c); (a) and (b) are addressed by design (fixed effort, fixed prompts) and by the engine docs. Primary-source facts for (a) and (b) are in section 4.

Cost model (Avid): "daily cost = ticks × triage + hits × (conductor + worker + verifier). At real prices (triage $0.01, conductor $0.35, worker $0.10, verifier $0.40): a daily janitor is ~$2.56/day; a 15-minute babysitter with Fable in the triage seat is $34/day for the identical outcome." (The Substack's public preview makes the same point: "Drive Fable 5 by hand, and a heavy session runs around $7 … one day it can burn $400–600 … the day costs a few dollars.")

`loop/scripts/log-cost.sh`:

```bash
#!/usr/bin/env bash
echo -e "$(date -Is)\t$1\t$2" >> "$(dirname "$0")/../memory/usage.log"
```

`loop/scripts/cost-check.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
F="$(dirname "$0")/../memory/usage.log"; touch "$F"; TODAY=$(date +%F)
case "${1:-}" in
  --budget)
    spent=$(awk -F'\t' -v d="$TODAY" '$1 ~ d {s+=$3} END{printf "%.2f",s}' "$F")
    awk -v s="$spent" -v b="$2" 'BEGIN{exit (s>=b)?1:0}' \
      || { echo "spent \$$spent of \$$2" >&2; exit 1; };;
  --report)
    awk -F'\t' -v since="$(date -d '7 days ago' +%F)" \
      '$1>=since{s[$2]+=$3;t+=$3} END{for(k in s) printf "  %-10s $%.2f\n",k,s[k]; printf "  TOTAL      $%.2f\n",t}' "$F";;
esac
```

Usage log format: `memory/usage.log`, columns `timestamp(ISO)<TAB>stage<TAB>usd`. Note the costs logged are the flat estimates hard-coded in `loop.sh` (0.01 / 0.35 / 0.10 / 0.40), not measured spend.

Rules: "cadence is a cost decision (halving the interval doubles the floor); the quiet tick must cost cents; effort never above high in loops (xhigh is for one-shot reviews; effort is per step, not per run)."

**CHECK 6:** "After a week of ticks, ./scripts/cost-check.sh --report matches the formula within noise. Set DAILY_BUDGET_USD to a number you would not mind wasting."

### Layer 7: Prompt-injection defence

**Not present in any public source.** The Substack preview says only: "needed because the loop reads issues and logs written by strangers, and strangers can hide instructions in them." What the public builds do that bears on it: the triage prompt forbids actions ("No fixes, no opinions"); untrusted text (issues, CI logs) only ever reaches the one-cent triage model, whose output is a findings list; the conductor is read-only and sees STATE/TRUST/CONTRACT; the worker sees only the JSON work order; the verifier sees only SPEC and DIFF with no tools; "anything requests a secret" is a wake-up in `contract.md`; and the reroute check refuses to iterate on output from a model that was not chosen. Treat anything beyond that as unknown.

### Layer 8: Runbook

Quoted from Avid ("The runbook. What each alarm means and what to do"):

| Signal | Meaning | Action |
|---|---|---|
| exit 2 (reroute) | safeguard router swapped models mid-run | read the checkpoint; re-run item tomorrow; never iterate on the swapped output |
| stop_reason: refusal | safety classifier declined (HTTP 200, not an error) | official path: fallback to Opus 4.8 via fallbacks or middleware; if it recurs on one skill, audit that skill for reasoning-echo or cyber/bio-adjacent phrasing |
| exit 3 (budget) | daily spend hit the line | make audit; find which stage grew; fix the effort map, not the budget |
| ALERT demoted | an established skill dropped below 90% | read its last 3 fails; usually the spec pattern, not the worker |
| goal VIOLATED | something finished stopped being true | sentinel gives suspects; fix goes through the normal pipeline |
| maker/checker standoff x2 | neither is presumed right | you decide, or run the standoff-breaker (third fresh reviewer, judges evidence, may not split the difference) |
| verify-goals timeout | predicate too expensive | that is a violation; cheapen the predicate |

Ops glue (BUILD 8), `Makefile`:

```makefile
tick:            ; ./loop/loop.sh
queue:           ; @grep -E "review:|queued:|FAILED:|rerouted" loop/memory/STATE.md || echo empty
trust:           ; @./loop/scripts/trust-log.sh --render
audit:           ; @./loop/scripts/cost-check.sh --report
goals:           ; @./loop/verify-goals.sh
clean-worktrees: ; @git worktree list | awk '/wt-/{print $$1}' | xargs -rn1 git worktree remove --force
```

Cron (enabled at the start of week 2):

```
0 7 * * 1-5  cd /path/to/repo/loop && ./loop.sh >> memory/cron.log 2>&1
30 7 * * *   cd /path/to/repo/loop && ./verify-goals.sh >> memory/cron.log 2>&1
```

### Layer 9: 30-day rollout plan

Quoted from Avid ("Do not skip graduations; each unlocks the next"):

| Week | Level | You do | Graduate when |
|---|---|---|---|
| 1 | L1 report | Builds 1-6; make tick by hand daily; read everything | 3 consecutive runs route exactly as you would have |
| 2 | L2 draft | cron on; make queue with coffee; reviews feed the ledger | 2 skills cross 20 logged runs |
| 3 | L3 ship | make audit vs the formula; best skill goes unattended | 1 week, zero interventions |
| 4 | L4 grow | compost signs-offs; approve 1 proposed skill; run the delete pass | you removed something and nothing broke |

### Optional loops (Avid BUILD 7; no Substack counterpart in the public list)

Each has an install condition. Quorum (three cheap models vote before waking Fable; wake on 2 of 3), Ratchet (`/goal` that may only move a metric one way, with "never edit lint config to lower it"), Sparring (daily builder/breaker `/loop` pair; breaker writes one failing test under `tests/sparring/`, builder may only fix code), Compost (weekly `/loop 7d` that reads the week's failures and proposes at most 3 changes: a CLAUDE.md law, a skill fix, or a missing standing goal; propose only). The prompt texts are in the Avid article; I have not copied them here because they are not part of the nine layers.

## 4. Primary sources behind the engine claims

Everything in this section is quoted from Anthropic's documentation as fetched on 2026-10-19.

### Model facts — https://platform.claude.com/docs/en/models/fable-5/overview

- "**Legacy.** Released June 9, 2026." "Model ID: `claude-fable-5`" "Context window: 1M tokens · Max output: 128K tokens · Input pricing: $10 / MTok · Output pricing: $50 / MTok". Claude Fable 5.1 is now the current Fable model.
- Pricing table: 5m cache write $12.50 / MTok; 1h cache write $20 / MTok; **Cache read $1 / MTok**; Batch API 50% discount. (This is the "prompt caching" leg of the budget: cached reads are 10% of input price.)
- Thinking: "Adaptive (always on)". Default effort: `high`. Reliable knowledge cutoff: Jan 2026.

### Effort — https://platform.claude.com/docs/en/build-with-claude/effort

- "`xhigh` | Extended capability for long-horizon work … | Long-running agentic and coding tasks (over 30 minutes) with token budgets in the millions".
- "Effort is the primary control for trading off intelligence, latency, and cost on Claude Fable 5. **Start with `high`, the default, for most tasks**, use `xhigh` for the most capability-sensitive workloads, and step down to `medium` or `low` for routine work. Lower effort settings on Claude Fable 5 still perform well and often exceed `xhigh` performance on prior models. At `high` and `xhigh`, set a large `max_tokens`. It's a hard limit on total output (thinking plus response text)."
- Per-message effort change (cache-preserving) is **not** available on Claude Fable 5 (it is on 5.1): "models without per-message effort, including Claude Fable 5, return a 400 error".
- "Hold top-level effort constant within cached conversations: Changing the top-level effort value between requests invalidates prompt caching".

### Prompting guide — https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5

The official prompt blocks the article's "prompt pack" abridges. Official wording:

Anti-overplanning:
> When you have enough information to act, act. Do not re-derive facts already established in the conversation, re-litigate a decision the user has already made, or narrate options you will not pursue in user-facing messages. If you are weighing a choice, give a recommendation, not an exhaustive survey. This does not apply to thinking blocks.

Anti-gold-plating:
> Don't add features, refactor, or introduce abstractions beyond what the task requires. A bug fix doesn't need surrounding cleanup and a one-shot operation usually doesn't need a helper. Don't design for hypothetical future requirements: do the simplest thing that works well. Avoid premature abstraction and half-finished implementations. Don't add error handling, fallbacks, or validation for scenarios that cannot happen. Trust internal code and framework guarantees. Only validate at system boundaries (user input, external APIs). Don't use feature flags or backwards-compatibility shims when you can just change the code.

Grounded progress claims ("In Anthropic's testing, this nearly eliminated fabricated status reports even on tasks designed to elicit them" — this is the "Anthropic says so itself" line in the Substack preview):
> Before reporting progress, audit each claim against a tool result from this session. Only report work you can point to evidence for; if something is not yet verified, say so explicitly. Report outcomes faithfully: if tests fail, say so with the output; if a step was skipped, say that; when something is done and verified, state it plainly without hedging.

Autonomous pipeline reminder:
> You are operating autonomously. The user is not watching in real time and cannot answer questions mid-task, so asking "Want me to…?" or "Shall I…?" will block the work. For reversible actions that follow from the original request, proceed without asking. Offering follow-ups after the task is done is fine; asking permission after already discussing with the user before doing the work is not. Before ending your turn, check your last paragraph. If it is a plan, an analysis, a question, a list of next steps, or a promise about work you have not done ("I'll…", "let me know when…"), do that work now with tool calls. End your turn only when the task is complete or you are blocked on input only the user can provide.

Memory format:
> Store one lesson per file with a one-line summary at the top. Record corrections and confirmed approaches alike, including why they mattered. Don't save what the repo or chat history already records; update an existing note rather than creating a duplicate; delete notes that turn out to be wrong.

Verifier ("Separate, fresh-context verifier subagents tend to outperform self-critique."):
> Establish a method for checking your own work at an interval of [X] as you build. Run this every [X interval], verifying your work with subagents against the specification.

Other facts the article leans on:
- Long turns: "Individual requests on hard tasks can run for many minutes at higher effort settings … and autonomous runs can extend for hours … consider restructuring harnesses to check on runs asynchronously, for example through scheduled jobs, rather than blocking."
- Safety classifiers: "Claude Fable 5 runs safety classifiers that target offensive cybersecurity techniques …, biology and life sciences content …, and extraction of the model's summarized thinking. Benign cybersecurity work and beneficial life sciences tasks may also trigger these safeguards. To re-route declined requests automatically, configure server-side or client-side fallback to Claude Opus 4.8."
- Reasoning echo: "Prompts, skills, or harness instructions that tell the model to echo, transcribe, or explain its internal reasoning as response text may trigger the `reasoning_extraction` refusal category on Claude Fable 5. Server-side fallback with `fallbacks: "default"` doesn't retry requests declined with `reasoning_extraction`".
- Old skills: "Skills developed for prior models are often too prescriptive for Claude Fable 5 and can degrade output quality."

### Refusals — https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback and https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons

- "Claude declined to generate a response. Safety classifiers return this stop reason as a normal HTTP 200 response, not an error."
- Response shape: `"stop_reason": "refusal"`, `"stop_details": {"type": "refusal", "category": "cyber", "explanation": "…"}`; `content` is empty; `stop_details` is `null` for every other stop reason.
- Categories: `cyber`, `bio`, `frontier_llm`, `reasoning_extraction`, `general_harms`. `reasoning_extraction` is billed even before any output and "has no recommended fallback model, so change the prompt rather than retry".
- Server-side fallback: `fallbacks: "default"` with beta header `server-side-fallback-2026-07-01`; not available on Bedrock / Google Cloud / Foundry or the Batch API. The SDK examples detect which model served the response by pairing `response.model` with `stop_reason` (`served_by_fallback = fallback_ran and stop_reason != "refusal"`). This is the API-level equivalent of the article's "never iterate on a model you didn't choose".

### Task budgets — https://platform.claude.com/docs/en/build-with-claude/task-budgets

- Beta; header `task-budgets-2026-03-13`; supported on Claude Fable 5. Shape: `output_config.task_budget: {"type": "tokens", "total": N, "remaining": M?}`; minimum `total` 20,000.
- "Task budgets are a **soft hint, not a hard cap** … The enforced limit on total output tokens is still `max_tokens`".
- "A budget that is too small for the task can cause refusal-like behavior."
- "**Task budgets are not supported on Claude Code or Cowork surfaces.** Use task budgets directly through the Messages API." See section 5.

### Claude Code CLI — https://code.claude.com/docs/en/cli-reference and https://code.claude.com/docs/en/headless

- `--effort`: "Options: `low`, `medium`, `high`, `xhigh`, `max`, or `ultracode`."
- `--allowedTools`: "Tools that execute without prompting for permission … **To restrict which tools are available, use `--tools` instead.**"
- `--tools`: "Restrict which built-in tools Claude can use. Use `""` to disable all, `"default"` for the default set, or tool names like `"Bash,Edit,Read"`."
- `--max-budget-usd`: "Stop the run once estimated spend on API calls reaches this amount (print mode only) … Spend can pass the cap, so leave headroom."
- `--output-format json`: "structured JSON with result, session ID, and metadata"; "the response payload includes `total_cost_usd` and a per-model cost breakdown". The field name `modelUsage` used in `loop.sh` is not named on the page.
- Unattended runs: "In a `-p` run with no host, these requests are denied either way" and `--permission-prompts none` (v2.1.259+) tells Claude not to retry; `--permission-mode dontAsk` "denies every call that would otherwise prompt … Actions that need no approval in Manual mode still run, such as file reads in your working directories and the read-only command set". `--bare` is "the recommended mode for scripted and SDK calls".
- `/loop [interval] [prompt]`: "Run a prompt repeatedly while the session stays open." `/goal [condition|clear]`: "Claude keeps working across turns until the condition is met". `/usage-credits`: opens usage-credits billing settings (where the "usage credits cap" in the prerequisites is set). — https://code.claude.com/docs/en/commands

### Platform timeline the preview refers to — https://en.wikipedia.org/wiki/Claude_Mythos

- Released June 2026 as a "Mythos-class" model; flagged cyber/bio/distillation requests "handled by the less capable Claude Opus".
- June 12: US government letter "prohibiting access to both Mythos 5 and Fable 5 to any non-U.S. national whatsoever"; "Anthropic revoked access to both models for all customers." June 30: restrictions lifted; access restored July 1. That is 19 days, matching the preview's "switched off worldwide for 19 days".
- Subscriptions: "temporary access to Fable 5 from July 1 until July 19, extended several times"; "On July 20, Fable 5 was made a standard feature of Anthropic's higher-tier subscription plans." The preview (written July 10) says inclusion ends July 12 and then bills via credits; per Wikipedia that cutoff was extended and then dropped for higher tiers. The usage-credits billing note in the paywalled half is therefore partly stale.

## 5. Where the shipped scripts and the primary sources disagree

These matter if any of this is adopted in this repo.

1. **`--allowedTools` does not restrict tools.** The CLI reference says it only auto-approves; `--tools` restricts availability. In a `-p` run with no permission host, unlisted tool calls are denied, so the conductor cannot *edit* with `--allowedTools "Read"`, but a verifier run with `--allowedTools ""` can still read files in the working directory and run the read-only command set without approval. For a verifier that truly sees "only spec + diff", use `--tools ""` (and `--bare`, which also stops CLAUDE.md, hooks and MCP servers from loading into the verifier's context). The Substack's "physically unable to change files" wording is defensible for the conductor; "no tools" for the verifier is not what the flag does.
2. **Task budgets cannot be used through `claude -p`.** The Substack's second budget leg ("Anthropic's task-budget feature") only applies if the conductor is called through the Messages API, not via Claude Code. The Claude Code equivalent is `--max-budget-usd`, which the scripts do not use. Also, task budgets are advisory; `max_tokens` is the hard cap.
3. **No `max_tokens` flag exists on the CLI.** CHECK 0 hedges with "where supported"; the scripts never set it.
4. **xhigh in a loop contradicts the constitution.** CLAUDE.md says "Never exceed effort high inside any loop. xhigh is for one-shot reviews only", and BUILD 6 says "effort never above high in loops", yet `loop.sh` runs the conductor at `--effort xhigh` inside the `for` loop. Pick one.
5. **`modelUsage` is unverified.** The headless docs promise a per-model cost breakdown in the JSON output but do not name the field. Check the actual key before relying on the reroute check (exit 2).
6. **Logged costs are constants, not measurements.** `log-cost.sh` records 0.01/0.35/0.10/0.40; the real figure is in the JSON output's `total_cost_usd`. The daily cap therefore caps *estimated* iterations, not dollars.
7. **GNU-only shell.** `date +%s%3N`, `date -d '7 days ago'`, `sed -i` without a suffix, `grep -P`, `xargs -r`, and `timeout` all assume GNU coreutils/grep. On macOS they need `coreutils`/`gnu-sed`/`ggrep` or rewriting.
8. **Fable 5 is now legacy.** Anthropic's overview says to migrate to Claude Fable 5.1 (released 2026-09-01 per Wikipedia). 5.1 adds cache-preserving per-message effort, which changes the "hold effort constant" advice.
9. **OpenRouter model IDs** (`deepseek/deepseek-v4-flash`, `moonshotai/kimi-k2.6`, `qwen/qwen-3.6`) were not verified.
10. **CHECK 1's "wc -l CLAUDE.md under 15"** conflicts with "under 150 lines" in the same build; almost certainly a typo for 150.

## 6. What is still unknown

- The paywalled text of Layers 1–9 in the Substack post itself, including whether it ships anything the free sources do not (a literal constitution checker, a prompt-injection layer, task-budget wiring, a usage-credits billing note).
- Ray Fu's Part 2 (steps 7–10), which is on Skool/Instagram only.
- Whether Linas's post derives from Avid's notes, or both from a common source. Avid's page is dated four days earlier and carries an "edited by Claude Opus 4.8 max" disclaimer.

## Sources

- Substack post (public preview and post JSON): https://linas.substack.com/p/claude-fable-5-agentic-os-guide ; https://linas.substack.com/api/v1/posts/claude-fable-5-agentic-os-guide ; Wayback 2026-09-25: http://web.archive.org/web/20260925095716/https://linas.substack.com/p/claude-fable-5-agentic-os-guide
- Avid (@Av1dlive), X article: https://x.com/Av1dlive/article/2074169173178212621 (Wayback 2026-07-18: http://web.archive.org/web/20260718215804/https://x.com/Av1dlive/article/2074169173178212621)
- Ray Fu, Part 1: https://www.raycfu.com/guides/agentic-os ; video: https://www.youtube.com/watch?v=3ujOdi9W1EI
- Anthropic: Fable 5 overview https://platform.claude.com/docs/en/models/fable-5/overview ; Prompting Claude Fable 5 https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5 ; Effort https://platform.claude.com/docs/en/build-with-claude/effort ; Task budgets https://platform.claude.com/docs/en/build-with-claude/task-budgets ; Refusals and fallback https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback ; Stop reasons https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons
- Claude Code: CLI reference https://code.claude.com/docs/en/cli-reference ; Headless https://code.claude.com/docs/en/headless ; Commands https://code.claude.com/docs/en/commands
- Timeline: https://en.wikipedia.org/wiki/Claude_Mythos
