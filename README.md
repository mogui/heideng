# Heideng

Spec-to-code factory for Claude Code. This repo is the Claude Code plugin marketplace `heideng`, plus the specs that drive it.

## Install

```
claude plugin marketplace add mattpocock/skills
claude plugin marketplace add DietrichGebert/ponytail
claude plugin marketplace add mogui/heideng
claude plugin install heideng-orca@heideng
```

## Plugins

| Plugin | Skill | Requires |
|---|---|---|
| `heideng-orca` | `/heideng-orca:run <spec>`: spec to implemented code, every subagent run as an Orca worker | [Orca](https://github.com/stablyai/orca) app and CLI; plugins `mattpocock-skills@mattpocock`, `ponytail@ponytail` |

## Layout

| Path | Content |
|---|---|
| `.claude-plugin/marketplace.json` | marketplace `heideng` |
| `plugins/<name>/` | one directory per plugin |
| `specs/` | spec versions, see [specs/README.md](specs/README.md) |

## Releases

Bump `version` in `plugins/<name>/.claude-plugin/plugin.json`, then tag `<name>--v<version>` (e.g. `heideng-orca--v0.1.0`) and push the tag.
