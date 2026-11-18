---
name: run
description: "Turn a spec into implemented code, unattended after one landing question: tickets then implementation, every subagent an Orca worker, skill questions answered by an Orca advisor worker."
argument-hint: "<path to spec, e.g. docs/SPEC.md>"
disable-model-invocation: true
---

# Heideng orca

Input: spec at `$ARGUMENTS` (default `docs/SPEC.md`).

You are the **coordinator** of an unattended factory. You follow [to-tickets.md](to-tickets.md) then [implement-spec.md](implement-spec.md): every subagent is an **Orca worker**, and every question a skill would ask the user goes to the **advisor** worker. The user hears from you at step 0 in the chat, then through the run's thread in the Plannotator Inbox ([Human inbox](#human-inbox)) and in status reports.

Interrupted run (new session, compacted context, user asks "where are we"): follow [RESUME.md](RESUME.md) before anything else.

## Preconditions

All hold before step 0; on a miss, stop and tell the user the fix.

- `orca --version` works and the `orchestration` skill is available (both ship with the Orca app). Load `orchestration` once and follow its run/worker/ask protocol for everything below.
- Plugin dependencies enabled: `mattpocock-skills` (`tdd`, `code-review`, `pr`), `ponytail` (`ponytail-review`), and `pstack` (`create-verification-skill`, `maintain-verification-skill`). Skill names below drop the plugin prefix.
- The `plannotator_inbox` tool is available (fix: `claude plugin marketplace add backnotprop/plannotator`, `claude plugin install plannotator@plannotator`, restart Claude Code). Run `plannotator inbox --help` once and follow it for block syntax, threads and replies.
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

Route every question (from a skill you run, or an implementer `ask`) with `orca orchestration send --to dispatch:<advisor root dispatch>`, then relay the answer verbatim with `orca orchestration reply --id <msg>`. The advisor answers one question per `worker_done`; between questions it shows `succeeded`/`unverifiable`, which is idle, and the next `send` reopens it. A `NEEDS USER` answer goes to the user as a question in the Inbox thread ([Human inbox](#human-inbox)); the advisor's pick is its `Recommended:` line.

Expect one question per ticket: `tdd` makes every implementer confirm its test seams.

Done: advisor dispatch id in `state.md`.

## 2. Tickets

Follow [to-tickets.md](to-tickets.md): draft the breakdown with the **skeleton** and the **verify ticket** `V`, quiz the advisor, publish.

Done: every ticket, `V` included, published with its blocking links and listed in `state.md` as `queued` or `human`.

## 3. Implement

Follow [implement-spec.md](implement-spec.md): integration branch, one implementer worker per frontier ticket, merge on each `worker_done`, `code-review` and `ponytail-review`, one fix worker.

Done: every non-`human` ticket `merged`, both reviews written, the fix worker's branch merged.

## 4. Final verify

Runs after the review fix is merged: the code it sees is the code that lands.

1. One **verifier** worker, `--worktree current`, runs `maintain-verification-skill` on the integration branch: every feature in the map driven live, no `pending` left. Drift in the map it fixes and commits; product gaps go in its report, each labelled with its acceptance criterion or as a regression against the baseline.
2. Each product gap becomes a fix ticket: published to the tracker, spec from the template with the gap's feature file, launched, and merged as in step 3.
3. Fix tickets merged: run the verifier again. At most 2 rounds; each gap still open goes to the Inbox thread as a question (fix it in a follow-up, or accept it as is), with the verifier's evidence attached. Record round and outcome in `state.md`.

Done: verifier outcome `clean`, or the round limit reached with every open gap listed.

## Human inbox

Everything the user must decide or do after step 0 goes to the Plannotator Inbox through the `plannotator_inbox` tool. The user answers there when they can; the run keeps going. Only the coordinator writes to the Inbox: workers keep `ask` and `worker_done`.

- **One thread per run.** Every `send_message` carries `thread: "<spec-slug> run"`. Record the returned `thread_id` in `state.md`.
- **Decision** (a `NEEDS USER` answer, a verify gap left after round 2): one `:::question` block per decision, written for someone who has not followed the run. It says what the choice affects in product terms, lists the choices, and carries `Recommended:`, `Stakes:`, and `Holds up: #N; #M` naming the tickets that wait. Add `Stopped:` when no ticket can advance without the answer. Mark the waiting tickets `asking` and keep working the rest of the frontier.
- **Human task** that blocks agent tickets (a login, a real account, a GUI step): one `:::action` block with `Where:` and `Holds up:`.
- **Answer.** It arrives as a new turn starting `Plannotator Inbox: <title> (<reply id>)`. Relay the pick to the waiting worker with `orca orchestration reply`, write it to `$F/notes/`, and confirm in the thread with `reply_to: <reply id>`. Questions the reply leaves open stay open: wait for them, in the Inbox only. Withdraw a question the run made moot with `supersedes`.
- **Answer after landing.** A decision that changes code becomes a fix ticket, implemented, verified and landed as steps 3, 4 and Landing describe. With a PR, rerun `export-decisions` (Landing) after every answer.

## Supervision

Silence is the failure mode: a worker waiting on an answer that never arrived looks identical to a worker working. The coordinator notices.

- **Watch loop.** After every `worker-start`, `reply`, or `send`, keep a Monitor armed over all active dispatches: poll `orca orchestration worker-show --dispatch <id>` every 30 s and emit a line when `state`, `stage` or `attention` changes; exit when all settle; re-arm on expiry. It reads state only, leaving mail for `check`. On each event run `check`.
- **Verify the wake.** About a minute after a `reply`, `worker-read --dispatch <id> --source auto`: new tool activity means it resumed. Still parked on its `ask`: deliver the answer with `send --to dispatch:<id>`, nudge with `orca terminal send --terminal <handle> --text "Your question <msg> was answered; run orca orchestration check and continue." --enter`, and confirm activity again.
- **Stalled worker** (no activity, no mail, `attention=input`): read its output, find what it waits on, unblock it the same way.
- **Trust but read.** Git output that looks truncated or contradictory may come from an output-filtering proxy (`rtk`): rerun it unfiltered (`rtk proxy git ...`) before acting.

## Ledger

Keep `$F/state.md` current at every milestone, one line per item: run id, advisor root dispatch, Inbox thread id, integration branch, landing decision, verify round and outcome, then per ticket `#N status dispatch branch sha` with status in `queued | running | asking | done | merged | human`. It is the cheap map [RESUME.md](RESUME.md) rebuilds from; the tracker, git, and Orca are the sources of truth.

## Landing and cleanup

1. Full test suite and app build green on the integration branch.
2. Land per step 0 (the `pr` skill writes the PR body; reference every agent ticket with `Closes #N`). Leave `ready-for-human` tickets and any parent/map issue open, and say so. With a PR: pass its ref as `pulls` on the final summary message, then run `plannotator inbox export-decisions --thread <thread id> --pr <ref>` so the user's decisions sit on the PR.
3. For each implementer worktree, confirm its branch is an ancestor of the integration branch, then `orca worktree rm --worktree path:<abs path> --delete-branch`. Release the advisor with `worker-release`.

## Reporting

Short status at each milestone: tickets published, `V` merged, each ticket merged, review done, each verify round, landed, cleaned up. Each names what runs now and what each worker waits on. Every merge report lists what could not be verified without a GUI or real accounts.

The final summary is one message in the Inbox thread: what landed, the questions still open, the `ready-for-human` tickets, and everything that could not be verified without a GUI or real accounts, with the verify evidence in `attachments`. The chat gets one line with the thread's `url`.
