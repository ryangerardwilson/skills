# skills

Shared skills for Claude Code, Grok Build, and Codex.

`main` on https://github.com/ryangerardwilson/skills is the source of truth.
Every skill lives once, at `plugins/skills/skills/<name>/SKILL.md`, inside one
plugin named `skills`. The `r2d2 skills` screen creates, edits, and deletes
skills and pushes each change to `main`. Hosts pick up a change when they
refresh this marketplace.

## Install

Claude Code:

```sh
claude plugin marketplace add ryangerardwilson/skills
claude plugin install skills@skills
```

Grok Build:

```sh
grok plugin marketplace add ryangerardwilson/skills
grok plugin install skills --trust
```

Grok keeps a plugin off until it is enabled. Add `skills` to `[plugins] enabled`
in `~/.grok/config.toml`, or run `grok plugin enable skills`.

Codex:

```sh
codex plugin marketplace add ryangerardwilson/skills
codex plugin add skills@skills
```

## Update

```sh
claude plugin marketplace update skills && claude plugin update skills@skills
grok plugin marketplace update && grok plugin update skills
codex plugin marketplace upgrade skills
```

## Layout

```
.claude-plugin/marketplace.json     Claude Code marketplace
.grok-plugin/marketplace.json       Grok Build marketplace
.agents/plugins/marketplace.json    Codex marketplace
plugins/skills/.claude-plugin/plugin.json
plugins/skills/.grok-plugin/plugin.json
plugins/skills/.codex-plugin/plugin.json
plugins/skills/skills/<name>/SKILL.md
```

A `SKILL.md` starts with YAML frontmatter that has `name` and `description`.
`name` equals the directory name: lowercase letters, digits, and hyphens, not
starting or ending with a hyphen.
