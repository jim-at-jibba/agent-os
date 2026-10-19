# Agent OS

The permanent layer of files, scripts, and ledgers that lets a model work unattended and be trusted to. The model supplies intelligence; the OS supplies honesty.

## Language

**Constitution**:
The short file of hard rules every run reads first. Each rule is checkable by a script; a rule with no number, no "never", and no verifying command is not a rule.
_Avoid_: Guidelines, system prompt, CLAUDE.md rules

**Heartbeat**:
One scheduled, unattended run of the OS against a workload, from planning through the final vote.
_Avoid_: Cron job, daily run, loop

**Workload**:
A target the heartbeat operates on, such as the vault. Each workload has its own verifiers and budget.
_Avoid_: Project, repo, target

**Planner**:
The party that decides what a heartbeat does. It cannot change files.
_Avoid_: Orchestrator, Janus

**Executor**:
The party that carries out the planner's decisions. Always a cheaper model than the planner.
_Avoid_: Worker, subagent

**Verifier**:
The independent party that judges whether executed work meets its acceptance criteria. Never the executor, never the same model family as the executor.
_Avoid_: Reviewer, judge, checker

**Vote**:
The final, scripted yes/no on a piece of work. A plain shell script, chosen because a script cannot be talked into anything.
_Avoid_: Gate, approval, sign-off

**Run log**:
The append-only record of every heartbeat: what was planned, executed, verified, voted, and spent.
_Avoid_: Trust ledger (a later layer built on the run log), audit log

**Budget stop**:
The hard daily spending line enforced by a script. Spending halts at the line regardless of what any model says.
_Avoid_: Cap, limit, quota

**Runbook**:
The map from every alarm the OS can raise to the exact human response.
_Avoid_: Playbook, incident guide
