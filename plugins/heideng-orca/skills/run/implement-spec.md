# Implement spec

The spec has published tickets ([to-tickets.md](to-tickets.md)). The goal is the entire spec implemented on a single **integration branch**.

The tickets are not a list of steps. They are a **task graph** with blocking relationships between them. This means there is always a **frontier**: the tickets whose blockers are all merged.

Every subagent is an Orca worker with a fresh context window. Communication to and from workers is sparse: pass **context pointers** (spec, tickets, `$F/notes/`, previous commits) and don't duplicate what they point to.

## Steps

1. Read the spec and tickets to understand the task graph.

2. Create the integration branch: `git switch -c <spec-slug>-impl` off the current branch; commit any pending agent-config files on it.

3. (optional) Start an **exploration worker**, `--worktree current`, for the exploration the tickets need: relevant codebase files or external documentation. It writes its markdown notes to `$F/notes/`, so implementers focus on implementation.

4. Build `$F/specs/<N>.md` for each ticket from [implementer-template.md](implementer-template.md), filling every `__PLACEHOLDER__`. `__FEATURE_FILES__` lists the feature files of the acceptance criteria the ticket covers; for the skeleton, which runs before `V`, drop rule 4. `V` uses `$F/specs/V.md` as written by to-tickets.

5. Launch every frontier ticket in parallel, one **implementer worker** per ticket, `V` included: `--worktree new-top-level --repo id:<repo> --name <slug>-t<N> --base-branch <integration> --spec-file $F/specs/<N>.md`.

6. On each `worker_done`: merge the branch (see Merging), recompute the frontier, launch the newly unblocked tickets. `V`'s edits to `.claude/skills/verify/` merge like any ticket.

7. Once every agent ticket is merged, start two read-only **review workers** in parallel on the integration branch vs its base, `--worktree current`: one runs `code-review`, one runs `ponytail-review`. Each writes `$F/review/<skill>.md`. They may start while only docs-only tickets remain.

8. One implementer worker (new worktree) fixes the findings of both review files; where they conflict, correctness and the spec win over cutting code. Merge its branch.

## Merging

- Implementer already merged the integration tip (`git merge-base --is-ancestor <integration> <branch>`): `git merge --ff-only`.
- Branch diverged but merges clean: `git merge --no-ff`, then build + tests on the integration branch.
- Conflicts, or several diverged branches at once: one merger worker in the integration branch's worktree; queue extra branches to it with `send`. Its spec names the **hotspot** (the shared seam both tickets touched, from their notes) and how to combine it.
- Implementer branch edits `.claude/skills/verify/` outside its `__FEATURE_FILES__`: route the diff to the advisor before merging.
- A merge changed a seam a running implementer touches: `send` that implementer a one-paragraph heads-up pointing at the note.
