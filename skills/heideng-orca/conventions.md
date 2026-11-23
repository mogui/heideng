# Code conventions

`docs/agents/code-conventions.md`, in the home repo, lists the project's code conventions. One entry per convention: **category** (tests, exceptions, observability, ...), **applies to** (stack, repo, or kind of work), **form** with its target:

| Form | Target | The worker |
|---|---|---|
| skill | a skill declared in `souk.yaml`, so present in every worktree | invokes it with the `Skill` tool |
| file | a path inside the home repo | reads it |
| link | a URL | opens it |
| criteria | the rule text itself | applies it |

A convention that lives in a skill outside `souk.yaml` (local to the home repo) is recorded as files: its `SKILL.md` and the documents it points to for that category. A project without the file runs without conventions.

## Proposing them (step 0, file missing)

Search the spec and the project (`CLAUDE.md`/`AGENTS.md` of every repo, the home repo's skills, docs), propose the entries you find, and ask the user which work each applies to. With the answer, in the home repo's integration worktree: write `docs/agents/code-conventions.md` and a line pointing to it in the `## Agent skills` block, one commit.
