---
status: accepted
---

# Build the OS in TypeScript on Mastra, not shell scripts and CLIs

The source article builds its agentic OS as shell scripts, cron, and vendor CLIs. We build it as a TypeScript application on Mastra instead, because Mastra's workflows, scorers, model routing, TokenCostControl processor, and per-call usage tracking cover the heartbeat, verifier, routing, budget stop, and run log layers directly, and because the user prefers to work in JavaScript. The cost is a dependency whose releases move quickly and an application that is less inspectable than a handful of scripts.

## Consequences

- The **vote** is a deterministic workflow step with no model in it, or a script outside Mastra. A scorer judges; it never votes.
- **Permission separation** is enforced by giving the planner agent no write tools. A script proves this on every run, since the enforcement lives in code rather than the OS.
- **Constitution checks** run outside Mastra so a dependency bump cannot silently disable them.
- The **scheduler** is still launchd or cron; Mastra does not start itself.
- The **planner** runs through the Codex CLI on the user's ChatGPT subscription, invoked as a workflow step that spawns the CLI, not as a Mastra model provider. Mastra's model routing therefore covers the executor and verifier only.
- External parties sit behind one **adapter interface** per kind: spawned CLIs (Codex, Pi, Claude Code) and OpenAI-compatible endpoints (Hermes). Each task declares its adapter and model. A new harness is one adapter and no heartbeat change.
- Hermes Agent, if adopted as executor, is reached through its OpenAI-compatible API server as an ordinary model endpoint.
- The llm CLI verifier research is superseded for the verifier role; its model pricing and structured-output findings remain useful.
