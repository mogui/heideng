# Implement spec

The spec has published tickets ([to-tickets.md](to-tickets.md)). The goal is the entire spec implemented on one **integration branch** `<spec-slug>-impl`, with the same name in every repo of `repos.md` (SKILL.md, Repos).

The tickets are not a list of steps. They are a **task graph** with blocking relationships between them. This means there is always a **frontier**: the tickets whose blockers are all merged.

Every subagent is an Orca worker with a fresh context window. Communication to and from workers is sparse: pass **context pointers** (spec, tickets, `$F/notes/`, previous commits) and don't duplicate what they point to.

## Steps

1. Read the spec and tickets to understand the task graph.

2. Every repo's **integration worktree** exists on `<spec-slug>-impl` (SKILL.md, step 0), its path in `state.md`. Merges, builds and tests of a repo happen there (`git -C <path>`); the main checkouts never change branch.

3. (optional) Start an **exploration worker**, `--worktree current`, for the exploration the tickets need: relevant codebase files of every repo or external documentation. It writes its markdown notes to `$F/notes/`, so implementers focus on implementation.

4. Build `$F/specs/<N>.md` for each ticket from [implementer-template.md](implementer-template.md), filling every `__PLACEHOLDER__`:
   - `__REPO__`, `__BUILD_CMD__`, `__TEST_CMD__` from the ticket's row in `repos.md`.
   - `__HOME_DIR__` the home repo's integration worktree; `__SPEC_PATH__` the spec inside it.
   - `__ADVISOR_DISPATCH__` the advisor root dispatch from `state.md`.
   - `__TICKET_READ_CMD__` the read operation of `docs/agents/issue-tracker.md`, written to work from any repo's worktree: the tracker named explicitly, or the ticket file's absolute path.
   - `__CHECKOUTS__` one line `<repo name>: <path>` per repo: the ticket's own repo says "this worktree", every other repo its integration worktree.
   - `__FEATURE_FILES__` the feature files of the acceptance criteria the ticket covers. Drop rule 5 for the skeleton, which runs before `V`, and for a ticket whose chain continues in another repo.
   - `__CONVENTIONS__` one line per entry of `docs/agents/code-conventions.md` ([conventions.md](conventions.md)) whose "applies to" matches the ticket's content: what the worker does and its target (invoke skill `<name>`, read `<absolute path in __HOME_DIR__>`, open `<url>`, apply `<rule text>`). No matching entry: drop the conventions block.

   `V` uses `$F/specs/V.md` as written by to-tickets.

5. Launch frontier tickets in parallel, one **implementer worker** per ticket, `V` included, at most 4 implementer workers running at once; the rest of the frontier waits for a free slot: a worktree ([Worktrees](SKILL.md#worktrees)) in the ticket's repo, name and branch `<slug>-t<N>`, from `<spec-slug>-impl`; then `worker-start --agent claude --worktree path:<path> --spec-file $F/specs/<N>.md`.

6. On each `worker_done`: check that it lists every convention of its `__CONVENTIONS__` as loaded; on a miss, `send` the worker the missing ones and wait for its next `worker_done`. Then merge the branch into its repo's integration worktree (see Merging), append the feature-file updates it reports to `$F/notes/verify-updates.md`, recompute the frontier, launch the newly unblocked tickets. `V`'s edits to `.claude/skills/verify/` merge like any ticket.

7. Once every agent ticket is merged, start two read-only **review workers** in parallel, `--worktree current`: one runs `code-review`, one runs `ponytail-review`, each on the integration branch vs its base in every repo the tickets touched (paths from `state.md`). Each writes `$F/review/<skill>.md`: `code-review` one section per repo, `ponytail-review` one section per [component](SKILL.md#components). They may start while only docs-only tickets remain.

   With more than one component, the `ponytail-review` worker runs one subagent per component in parallel, each running `ponytail-review` on that component's diff, and merges their findings into its file.

8. One implementer worker per repo with findings (a worktree in that repo as in step 5, in parallel) fixes that repo's sections of both review files; where they conflict, correctness and the spec win over cutting code. Merge each branch.

## Merging

All commands run in the integration worktree of the ticket's repo.

- Implementer already merged the integration tip (`git merge-base --is-ancestor <spec-slug>-impl <branch>`): `git merge --ff-only`.
- Branch diverged but merges clean: `git merge --no-ff`, then that repo's build + tests.
- Conflicts, or several diverged branches of one repo at once: one merger worker with `--worktree path:<integration worktree>`; queue extra branches of that repo to it with `send`. Its spec names the **hotspot** (the shared seam both tickets touched, from their notes) and how to combine it.
- A branch other than `V`'s edits `.claude/skills/verify/`: route the diff to the advisor before merging.
- A merge changed a seam a running implementer touches, in any repo: `send` that implementer a one-paragraph heads-up pointing at the note.
