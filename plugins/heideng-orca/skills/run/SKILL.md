---
name: run
description: "Turn a spec into implemented code, unattended after one landing question: /to-tickets then /implement-spec, every subagent an Orca worker, skill questions answered by an Orca advisor worker."
argument-hint: "<path to spec, e.g. docs/SPEC.md>"
disable-model-invocation: true
---

# Heideng orca

Input: spec at `$ARGUMENTS` (default `docs/SPEC.md`).

You are the **coordinator** of an unattended factory. You run Matt Pocock's `/to-tickets` then `/implement-spec`, with one substitution: every subagent those skills ask for is an **Orca worker**, and every question a skill would ask the user goes to the **advisor** worker. The user hears from you only at step 0, on `NEEDS USER`, and in status reports.

Interrupted run (new session, compacted context, user asks "where are we"): follow [RESUME.md](RESUME.md) before anything else.

## Preconditions

All hold before step 0; on a miss, stop and tell the user the fix.

- `orca --version` works and the `orchestration` skill is available (both ship with the Orca app). Load `orchestration` once and follow its run/worker/ask protocol for everything below.
- Plugin dependencies enabled: `mattpocock-skills` (`to-tickets`, `implement-spec`, `tdd`, `code-review`, `pr`) and `ponytail` (`ponytail-review`). Skill names below drop the plugin prefix.
- `CLAUDE.md` or `AGENTS.md` has an `## Agent skills` block (fix: `/setup-matt-pocock-skills`).
- Repo id from `orca repo list --json`. Build and test commands from the repo (README, package manifest, scripts), or from the spec when there is no code yet.

**Scratch dir** `$F` = `/tmp/<repo dir name>-factory/`, with `tickets/`, `specs/`, `notes/`, `review/`, and `state.md` (see Ledger). Workers run in other worktrees: every scratch path you hand to a worker or a CLI flag is absolute.

## 0. Landing decision

Ask the user once how finished work lands: push the integration branch and open a PR that closes the tickets (recommended), merge to the base branch, or keep it local. This is the only outward-facing step; settling it now keeps the end of the run unattended. Done: decision written to `state.md`.

## 1. Advisor

Start one long-lived worker that answers skill questions on the user's behalf:

```
orca orchestration worker-start --agent claude --worktree current --keep-open \
  --task-title "Product advisor" --spec "<advisor spec>" --json
```

Advisor spec: read-only; read the spec, the glossary/ADRs, and (once published) the tickets. For each question received, reply with `worker_done` whose body is the answer, labelled with the ticket: decisive, cites the spec section, picks the simplest option consistent with the spec when it is silent. Prefix `NEEDS USER` only for irreversible or outward-facing decisions.

Route every question (from a skill you run, or an implementer `ask`) with `orca orchestration send --to dispatch:<advisor root dispatch>`, then relay the answer verbatim with `orca orchestration reply --id <msg>`. The advisor answers one question per `worker_done`; between questions it shows `succeeded`/`unverifiable`, which is idle, and the next `send` reopens it. Forward only `NEEDS USER` answers to the user.

Expect one question per ticket: `tdd` makes every implementer confirm its test seams.

Done: advisor dispatch id in `state.md`.

## 2. Tickets

Load `to-tickets` and follow it. Write the draft breakdown to `$F/tickets/draft.md`. Its quiz (granularity, blocking edges, merge/split, spec coverage) goes to the advisor, which writes the verdict to `$F/tickets/review.md`. Apply the changes, then publish to the tracker with native blocking links (older `gh` lacks `--blocked-by`: use the dependencies API from `docs/agents/issue-tracker.md`). Tickets needing a real login or GUI get `ready-for-human` and stay outside the implement graph.

Done: every ticket published with its blocking links and listed in `state.md` as `queued` or `human`.

## 3. Implement

Load `implement-spec` and follow it, with these mappings:

| implement-spec says | Do |
|---|---|
| integration branch | `git switch -c <spec-slug>-impl` off the current branch; commit any pending agent-config files on it |
| exploration subagent | Orca worker, `--worktree current`, writes notes to `$F/notes/` |
| implementer subagent | Orca worker per ticket: `--worktree new-top-level --repo id:<repo> --name <slug>-t<N> --base-branch <integration> --spec-file $F/specs/<N>.md` |
| merger subagent | see Merging |
| code-review | two read-only review workers in parallel on the integration branch vs its base, `--worktree current`: one runs `code-review`, one runs `ponytail-review`; each writes `$F/review/<skill>.md` |
| code-review fixes | one implementer worker (new worktree) fixes the findings of both review files; where they conflict, correctness and the spec win over cutting code |

Build each `$F/specs/<N>.md` from [implementer-template.md](implementer-template.md), filling every `__PLACEHOLDER__`.

Launch every **frontier** ticket (all blockers merged) in parallel. On each `worker_done`: merge, recompute the frontier, launch the newly unblocked tickets. The reviews may start while only docs-only tickets remain.

Done: every non-`human` ticket `merged`, both reviews written, the fix worker's branch merged. Then Landing and cleanup.

### Merging

- Implementer already merged the integration tip (`git merge-base --is-ancestor <integration> <branch>`): `git merge --ff-only`.
- Branch diverged but merges clean: `git merge --no-ff`, then build + tests on the integration branch.
- Conflicts, or several diverged branches at once: one merger worker in the integration branch's worktree; queue extra branches to it with `send`. Its spec names the **hotspot** (the shared seam both tickets touched, from their notes) and how to combine it.
- A merge changed a seam a running implementer touches: `send` that implementer a one-paragraph heads-up pointing at the note.

## Supervision

Silence is the failure mode: a worker waiting on an answer that never arrived looks identical to a worker working. The coordinator notices.

- **Watch loop.** After every `worker-start`, `reply`, or `send`, keep a Monitor armed over all active dispatches: poll `orca orchestration worker-show --dispatch <id>` every 30 s and emit a line when `state`, `stage` or `attention` changes; exit when all settle; re-arm on expiry. It reads state only, leaving mail for `check`. On each event run `check`.
- **Verify the wake.** About a minute after a `reply`, `worker-read --dispatch <id> --source auto`: new tool activity means it resumed. Still parked on its `ask`: deliver the answer with `send --to dispatch:<id>`, nudge with `orca terminal send --terminal <handle> --text "Your question <msg> was answered; run orca orchestration check and continue." --enter`, and confirm activity again.
- **Stalled worker** (no activity, no mail, `attention=input`): read its output, find what it waits on, unblock it the same way.
- **Trust but read.** Git output that looks truncated or contradictory may come from an output-filtering proxy (`rtk`): rerun it unfiltered (`rtk proxy git ...`) before acting.

## Ledger

Keep `$F/state.md` current at every milestone, one line per item: run id, advisor root dispatch, integration branch, landing decision, then per ticket `#N status dispatch branch sha` with status in `queued | running | asking | done | merged | human`. It is the cheap map [RESUME.md](RESUME.md) rebuilds from; the tracker, git, and Orca are the sources of truth.

## Landing and cleanup

1. Full test suite and app build green on the integration branch.
2. Land per step 0 (the `pr` skill writes the PR body; reference every agent ticket with `Closes #N`). Leave `ready-for-human` tickets and any parent/map issue open, and say so.
3. For each implementer worktree, confirm its branch is an ancestor of the integration branch, then `orca worktree rm --worktree path:<abs path> --delete-branch`. Release the advisor with `worker-release`.

## Reporting

Short status at each milestone: tickets published, each ticket merged, review done, landed, cleaned up. Each names what runs now and what each worker waits on. Every merge report lists what could not be verified without a GUI or real accounts; the final summary collects them.
