# Heideng as a Claude Code plugin - notes

Heideng ships as a Claude Code plugin that contains the engine. The engine keeps principle 3 by running every node as its own `claude -p` process with settings generated for its zone.

## Plugin layout

| Component | Content |
|---|---|
| `bin/heideng` | the engine of `SPEC.md`; on the Bash tool's `PATH` while the plugin is enabled |
| skill `/heideng:accept` | the accept session, run in the person's own session (accept runs outside the sandbox, with the person) |
| commands `/heideng:run`, `/heideng:resume`, `/heideng:status` | thin wrappers over `heideng`; read `report.md` and `events.jsonl` |
| skills | `to-tickets`, `tdd`, `diagnosing-bugs`, `code-review`, `ponytail`, `ponytail-review`; each node loads its own through `--plugin-dir` |

## Isolation stays in the engine

Facts from the Claude Code docs (sandboxing, permissions, sub-agents, hooks, plugins-reference):

| Mechanism | Fact |
|---|---|
| Plugin `settings.json` | only `agent` and `subagentStatusLine` take effect; `sandbox.*` and `permissions.deny` are dropped |
| Subagents | inherit the parent's permission rules; per-subagent sandbox filesystem rules are not documented; plugin subagents ignore `hooks` and `permissionMode` |
| Sandbox `denyRead` | OS-level (Seatbelt, bubblewrap) for Bash and its child processes; does not stop the Read tool |
| `permissions.deny` on Read | applies to Read, best-effort to Grep and Glob; does not stop subprocesses |
| PreToolUse hooks | can deny by path; not documented as a security boundary; a timed-out hook does not block; hooks run outside the sandbox |

Consequences:
- One sandbox per Claude Code session. Zones with different denials need different `claude -p` processes, each with `--settings` carrying both `sandbox.filesystem` and `permissions.deny` for the denied zones.
- Routing stays in deterministic code (principle 1); no LLM coordinator reads the holdout zone.
- Hooks serve observation only (progress, notifications), never enforcement.

## Constraint

`heideng run` writes the zones and spawns `claude` processes, so it runs outside the sandbox: from a terminal, or `! heideng run <spec>` in a session. A `/heideng:run` going through a sandboxed Bash tool inherits that sandbox.
