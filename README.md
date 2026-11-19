# Heideng

Spec-to-code factory for Claude Code. This repo is a `souk` registry, plus the specs that drive it.

## Install

Requires the [Orca](https://github.com/stablyai/orca) app and CLI, `souk`, and the Plannotator plugin:

```
claude plugin marketplace add backnotprop/plannotator
claude plugin install plannotator@plannotator
```

In the home repo (the one holding the spec):

```
souk init                       # when the repo has no souk.yaml
souk registry add heideng https://github.com/mogui/heideng
souk presets apply heideng
```

The `heideng` preset installs the `heideng-orca` skill and the skills its workers use. The run copies the home repo's `souk.yaml` and `souk.lock` into every worktree it creates and syncs them, so every worker in every repo has the same skills.

## Skills

| Skill | Use |
|---|---|
| `heideng-orca` | `/heideng-orca <spec>`: spec to implemented code across one or more repos, every subagent an Orca worker |

## Layout

| Path | Content |
|---|---|
| `skills/<name>/` | one directory per skill |
| `presets/heideng.yaml` | the `heideng` preset: `heideng-orca` and its dependencies |
| `specs/` | spec versions, see [specs/README.md](specs/README.md) |

## Releases

Tag `v<version>` and push the tag. Pin a release with `--ref v<version>` on the `heideng` registry alias.
