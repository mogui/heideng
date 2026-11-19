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
   - `__TICKET_READ_CMD__` works from any repo's worktree: tracker repo named explicitly (`gh issue view <N> -R <owner>/<home repo>`), or the ticket file's absolute path.
   - `__CHECKOUTS__` one line `<repo name>: <path>` per repo: the ticket's own repo says "this worktree", every other repo its integration worktree.
   - `__FEATURE_FILES__` the feature files of the acceptance criteria the ticket covers. Drop rule 4 for the skeleton, which runs before `V`, and for a ticket whose chain continues in another repo.

   `V` uses `$F/specs/V.md` as written by to-tickets.

5. Launch every frontier ticket in parallel, one **implementer worker** per ticket, `V` included: `--worktree new-top-level --repo id:<ticket's repo id> --name <slug>-t<N> --base-branch <spec-slug>-impl --spec-file $F/specs/<N>.md`.

6. On each `worker_done`: merge the branch into its repo's integration worktree (see Merging), append the feature-file updates it reports to `$F/notes/verify-updates.md`, recompute the frontier, launch the newly unblocked tickets. `V`'s edits to `.claude/skills/verify/` merge like any ticket.

7. Once every agent ticket is merged, start two read-only **review workers** in parallel, `--worktree current`: one runs `code-review`, one runs `ponytail-review`, each on the integration branch vs its base in every repo the tickets touched (paths from `state.md`). Each writes `$F/review/<skill>.md` with one section per repo. They may start while only docs-only tickets remain.

8. One implementer worker per repo with findings (new worktree in that repo, in parallel) fixes that repo's section of both review files; where they conflict, correctness and the spec win over cutting code. Merge each branch.

## Merging

All commands run in the integration worktree of the ticket's repo.

- Implementer already merged the integration tip (`git merge-base --is-ancestor <spec-slug>-impl <branch>`): `git merge --ff-only`.
- Branch diverged but merges clean: `git merge --no-ff`, then that repo's build + tests.
- Conflicts, or several diverged branches of one repo at once: one merger worker with `--worktree path:<integration worktree>`; queue extra branches of that repo to it with `send`. Its spec names the **hotspot** (the shared seam both tickets touched, from their notes) and how to combine it.
- A branch other than `V`'s edits `.claude/skills/verify/`: route the diff to the advisor before merging.
- A merge changed a seam a running implementer touches, in any repo: `send` that implementer a one-paragraph heads-up pointing at the note.
