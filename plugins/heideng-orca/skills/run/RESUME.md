# Resuming a heideng-orca run

The run outlives any one coordinator context. Scratch dir `$F` as in SKILL.md. Rebuild the picture from the sources of truth, reconcile, then continue the normal loop in SKILL.md. Done when every ticket has exactly one status and every active worker is watched.

## 1. Read the state

| Source | Gives |
|---|---|
| `$F/state.md` | last known run id, advisor dispatch, Inbox thread id, integration branch, landing decision, verify round and outcome, per-ticket status |
| `orca orchestration run-list` / `run-use --run <id>` | rebinds you as coordinator of the existing Run |
| tracker (`docs/agents/issue-tracker.md`) | every ticket, its blockers, open/closed |
| `git log --first-parent --oneline <integration>` (unfiltered) | tickets already merged |
| `orca orchestration worker-list --json` | dispatches still `ready`/in progress, liveness, attention |
| `orca orchestration check` then `inbox --terminal run:<run id> --limit 50` | unhandled reports and questions, and the mail already handled |
| `plannotator_inbox` `read_thread` with the thread id | questions and action items still waiting on the user, and answers that arrived while no session was listening |
| `$F/notes/*.md` | seams and agreed decisions per ticket |

## 2. Reconcile each ticket

- On the integration branch: `merged`.
- Branch exists, `worker_done` received, not on the integration branch: merge it (Merging in implement-spec.md).
- Dispatch live with an unanswered `question`: route it to the advisor; answered but parked: verify the wake (Supervision).
- Ticket `asking` on an Inbox question: answered in the thread: relay it (Human inbox in SKILL.md); still open: leave it waiting.
- Dispatch live and working: watch it.
- Dispatch gone without `worker_done`, branch has a ticket commit: build + tests on the branch; green: merge it; red: restart.
- Dispatch gone without `worker_done`, branch has no ticket commit: restart the ticket with a fresh implementer from `$F/specs/<N>.md`.
- Blockers all merged and no dispatch: frontier; launch it.
- All tickets merged, Final verify not `clean`: resume step 4 at the round recorded in `state.md`.

Never restart a ticket that is already merged or has a live working dispatch.

## 3. Continue

Rewrite `state.md`, arm the watch loop over every active dispatch, send the user one status table (merged / running / waiting / human) with the Inbox thread's `url`, then resume the loop.
