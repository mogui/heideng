---
name: heideng-orca
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
- `souk --version` works, and the home repo's `souk.yaml` and `souk.lock` pass `souk sync --check`. They declare this skill and the skills workers use: `tdd`, `code-review`, `pr`, `ponytail-review`, `create-verification-skill`, `maintain-verification-skill` (fix: `souk init` if there is no `souk.yaml`, then `souk registry add heideng https://github.com/mogui/heideng` and `souk presets apply heideng`).
- The `plannotator_inbox` tool is available (fix: `claude plugin marketplace add backnotprop/plannotator`, `claude plugin install plannotator@plannotator`, restart Claude Code). Run `plannotator inbox --help` once and follow it for block syntax, threads and replies.
- You run in the **home repo**: the one holding the spec, and whose `CLAUDE.md` or `AGENTS.md` has an `## Agent skills` block (fix: `/setup-matt-pocock-skills`).
- Every `esistente` repo of the spec's `## Repos` (see [Repos](#repos)), and the home repo, is in `orca repo list --json` (fix: `orca repo add --path <abs path>`).
- No repo other than the home repo has its own `souk.yaml` (fix: merge its declarations into the home repo's `souk.yaml` by hand).
- Each such repo with a `.claude/settings.local.json`: tell the user that workers do not see it (new worktrees lack git-ignored files); protections workers need belong in `~/.claude/settings.json`. A warning, not a stop.

**Scratch dir** `$F` = `/tmp/<home repo dir name>-factory/`, with `tickets/`, `specs/`, `notes/`, `review/`, and `state.md` (see Ledger). Workers run in other worktrees: every scratch path you hand to a worker or a CLI flag is absolute.

## Repos

The spec's `## Repos` section is the index of the repos the run touches: one line per repo with name, role in the system, and `esistente` or `nuovo`. The home repo is always in it, even with no code. A spec without the section is a single-repo run: the home repo is the only repo, same path through every step.

`repos.md`, beside the spec, is derived from that index at step 0 and committed on the home repo's integration branch. One line per repo: name, Orca id, absolute path of the main checkout, base branch, build command, test command. Build and test come from the repo (README, package manifest, scripts), or from the spec for a repo with no code yet. Tickets, workers, and [RESUME.md](RESUME.md) read it.

## Code conventions

`docs/agents/code-conventions.md`, in the home repo, lists the project's code conventions. One entry per convention: **category** (tests, exceptions, observability, ...), **applies to** (stack, repo, or kind of work), **form** with its target:

| Form | Target | The worker |
|---|---|---|
| skill | a skill declared in `souk.yaml`, so present in every worktree | invokes it with the `Skill` tool |
| file | a path inside the home repo | reads it |
| link | a URL | opens it |
| criteria | the rule text itself | applies it |

A convention that lives in a skill outside `souk.yaml` (local to the home repo) is recorded as files: its `SKILL.md` and the documents it points to for that category. A project without the file runs without conventions.

## Worktrees

Every worktree the run creates is ready before an agent runs in it:

1. `orca worktree create --repo id:<repo id> --name <name> --branch <branch> --base-branch <base> --setup skip --json`. Its path is `result.worktree.path`. Orca reuses `--branch` only when that branch is absent or still at the base; otherwise it creates `<branch>-2`, so check `result.worktree.branch` is `refs/heads/<branch>`.
2. Copy the home repo's `souk.yaml` and `souk.lock` into it, then `souk -C <path> sync --frozen`: every worker has every skill, in every repo.
3. A worker runs in it with `orca orchestration worker-start --agent claude --worktree path:<path> ...`, without `--repo`, `--name` or `--base-branch`, which Orca rejects for existing worktrees.

## 0. Landing and repos

Ask the user once, in one message:

- How finished work lands, for every repo alike: push each integration branch (recommended), merge to each base branch, or keep it local.
- For each `nuovo` repo: create its remote, or keep it local.
- When the spec has no `## Repos` but reads as several components (services, apps, libraries to create): the `## Repos` you propose. Rejected: single-repo run.
- When the home repo has no `docs/agents/code-conventions.md`: the project's [code conventions](#code-conventions). Search the spec and the project (`CLAUDE.md`/`AGENTS.md` of every repo, the home repo's skills, docs), propose the entries you find, and ask which work each applies to.

This is the only outward-facing step; settling it now keeps the end of the run unattended. Then:

1. Each `nuovo` repo: a sibling directory of the home repo, `git init`, one empty commit on `main`, `orca repo add --path <abs path>`; its remote, if approved.
2. Each repo: append to `$(git -C <repo> rev-parse --git-common-dir)/info/exclude`, shared by all its worktrees, the lines missing among `souk.yaml`, `souk.lock`, and `.claude/skills/<name>/` for each resource in `souk.lock`. Workers then never commit the souk files.
3. Each repo: its **integration worktree** ([Worktrees](#worktrees)), name `<spec-slug>-impl-<repo name>`, branch `<spec-slug>-impl`, from its base branch. Copy pending agent-config files from the main checkout into it and commit them there.
4. In the home repo's integration worktree: the approved `## Repos` added to the spec, `repos.md` written, one commit.
5. Conventions answered: in the home repo's integration worktree, write `docs/agents/code-conventions.md` ([Code conventions](#code-conventions)) and a line pointing to it in the `## Agent skills` block, one commit.

From here on, "the spec" is the copy in the home repo's integration worktree; every path to it you hand a worker is that one.

Done: landing decision, remote decisions, and every integration worktree path written to `state.md`; `docs/agents/code-conventions.md` present when the project has conventions.

## 1. Advisor

Start one long-lived worker that answers skill questions on the user's behalf:

```
orca orchestration worker-start --agent claude --worktree current --keep-open \
  --task-title "Product advisor" --spec "<advisor spec>" --json
```

Advisor spec: read-only; read the spec, `repos.md`, the glossary/ADRs, and (once published) the tickets. For each question received, reply with `worker_done` whose body is the answer, labelled with the ticket: decisive, cites the spec section, picks the simplest option consistent with the spec when it is silent. Prefix `NEEDS USER` only for irreversible or outward-facing decisions.

Route every question (from a skill you run, or an implementer `ask`) with `orca orchestration send --to dispatch:<advisor root dispatch>`, then relay the answer verbatim with `orca orchestration reply --id <msg>`. The advisor answers one question per `worker_done`; between questions it shows `succeeded`/`unverifiable`, which is idle, and the next `send` reopens it. A `NEEDS USER` answer goes to the user as a question in the Inbox thread ([Human inbox](#human-inbox)); the advisor's pick is its `Recommended:` line.

Expect one question per ticket: `tdd` makes every implementer confirm its test seams.

Done: advisor dispatch id in `state.md`.

## 2. Tickets

Follow [to-tickets.md](to-tickets.md): draft the breakdown with the **skeleton** and the **verify ticket** `V`, quiz the advisor, publish.

Done: every ticket, `V` included, published with its blocking links and listed in `state.md` as `queued` or `human`.

## 3. Implement

Follow [implement-spec.md](implement-spec.md): one implementer worker per frontier ticket in its repo, merge on each `worker_done` into that repo's integration worktree, `code-review` and `ponytail-review` across repos, one fix worker per repo with findings.

Done: every non-`human` ticket `merged`, both reviews written, every fix worker's branch merged.

## 4. Final verify

Runs after the review fix is merged: the code it sees is the code that lands.

1. One **verifier** worker, `--worktree path:<home repo integration worktree>`, runs `maintain-verification-skill` with the system started from every repo's integration worktree: every feature in the map driven live, no `pending` left. It first applies `$F/notes/verify-updates.md` to the feature files. Drift in the map it fixes and commits; product gaps go in its report, each labelled with its acceptance criterion or as a regression against the baseline.
2. Each product gap becomes a fix ticket: published to the tracker, spec from the template with the gap's feature file, launched, and merged as in step 3.
3. Fix tickets merged: run the verifier again. At most 2 rounds; each gap still open goes to the Inbox thread as a question (fix it in a follow-up, or accept it as is), with the verifier's evidence attached. Record round and outcome in `state.md`.

Done: verifier outcome `clean`, or the round limit reached with every open gap listed.

## Human inbox

Everything the user must decide or do after step 0 goes to the Plannotator Inbox through the `plannotator_inbox` tool. The user answers there when they can; the run keeps going. Only the coordinator writes to the Inbox: workers keep `ask` and `worker_done`.

- **One thread per run.** Every `send_message` carries `thread: "<spec-slug> run"`. Record the returned `thread_id` in `state.md`.
- **Decision** (a `NEEDS USER` answer, a verify gap left after round 2): one `:::question` block per decision, written for someone who has not followed the run. It says what the choice affects in product terms, lists the choices, and carries `Recommended:`, `Stakes:`, and `Holds up: #N; #M` naming the tickets that wait. Add `Stopped:` when no ticket can advance without the answer. Mark the waiting tickets `asking` and keep working the rest of the frontier.
- **Human task** that blocks agent tickets (a login, a real account, a GUI step): one `:::action` block with `Where:` and `Holds up:`.
- **Answer.** It arrives as a new turn starting `Plannotator Inbox: <title> (<reply id>)`. Relay the pick to the waiting worker with `orca orchestration reply`, write it to `$F/notes/`, and confirm in the thread with `reply_to: <reply id>`. Questions the reply leaves open stay open: wait for them, in the Inbox only. Withdraw a question the run made moot with `supersedes`.
- **Answer after landing.** A decision that changes code becomes a fix ticket, implemented, verified and landed as steps 3, 4 and Landing describe. After landing, rerun the decisions export (Landing) after every answer.

## Supervision

Silence is the failure mode: a worker waiting on an answer that never arrived looks identical to a worker working. The coordinator notices.

- **Watch loop.** After every `worker-start`, `reply`, or `send`, keep a Monitor armed over all active dispatches: poll `orca orchestration worker-show --dispatch <id>` every 30 s and emit a line when `state`, `stage` or `attention` changes; exit when all settle; re-arm on expiry. It reads state only, leaving mail for `check`. On each event run `check`.
- **Verify the wake.** About a minute after a `reply`, `worker-read --dispatch <id> --source auto`: new tool activity means it resumed. Still parked on its `ask`: deliver the answer with `send --to dispatch:<id>`, nudge with `orca terminal send --terminal <handle> --text "Your question <msg> was answered; run orca orchestration check and continue." --enter`, and confirm activity again.
- **Stalled worker** (no activity, no mail, `attention=input`): read its output, find what it waits on, unblock it the same way.
- **Trust but read.** Git output that looks truncated or contradictory may come from an output-filtering proxy (`rtk`): rerun it unfiltered (`rtk proxy git ...`) before acting.

## Ledger

Keep `$F/state.md` current at every milestone, one line per item: run id, advisor root dispatch, Inbox thread id, integration branch, landing and remote decisions, per repo its integration worktree path, verify round and outcome, then per ticket `#N repo status dispatch branch sha` with status in `queued | running | asking | done | merged | human`. It is the cheap map [RESUME.md](RESUME.md) rebuilds from; the tracker, git, and Orca are the sources of truth.

## Landing and cleanup

1. In each repo's integration worktree: full test suite and build green.
2. Land per step 0, repo by repo in the order the tickets depend on each other:
   - Push: push the integration branch. When the project documents how to open a pull request (a forge CLI in `docs/agents/` or the README), open one per repo: the `pr` skill writes each body, references the other repos' PRs, and closes that repo's agent tickets with the syntax of `docs/agents/issue-tracker.md`. Otherwise the pushed branch is the deliverable: the final summary names each repo's branch, and a human opens the PRs. Pass every PR ref as `pulls` on the final summary message. Decisions: `plannotator inbox export-decisions --thread <thread id> --pr <ref>` for each GitHub (`owner/repo#N`) or GitLab (`group/project!N`) PR; otherwise `--out $F/decisions.md`, attached to the final summary.
   - Merge: `git merge --ff-only <spec-slug>-impl` in the repo's main checkout when it is on its base branch and clean; otherwise a `:::action` in the Inbox thread naming the repo and the command.

   Leave `ready-for-human` tickets and any parent/map issue open, and say so.
3. Cleanup. For each implementer worktree, in its repo, confirm its branch is an ancestor of the integration branch, then `orca worktree rm --worktree path:<abs path> --delete-branch`. Push or merge: remove each integration worktree with `orca worktree rm --worktree path:<abs path>`, keeping its branch. Local: keep the integration worktrees and list their paths in the final summary. Release the advisor with `worker-release`.

## Reporting

Short status at each milestone: tickets published, `V` merged, each ticket merged, review done, each verify round, landed, cleaned up. Each names what runs now and what each worker waits on. Every merge report lists what could not be verified without a GUI or real accounts.

The final summary is one message in the Inbox thread: what landed, the questions still open, the `ready-for-human` tickets, and everything that could not be verified without a GUI or real accounts, with the verify evidence in `attachments`. The chat gets one line with the thread's `url`.
