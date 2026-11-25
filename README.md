# skills

Shared skills for Claude Code, Grok Build, and Codex.

`main` on https://github.com/ryangerardwilson/skills is the source of truth.
Every skill lives once, at `plugins/r2d2-skills/skills/<name>/SKILL.md`, inside
one plugin named `r2d2-skills`. The marketplace name stays `skills`. The
`r2d2 skills` screen creates, edits, and deletes skills and pushes each change
to `main`. Hosts pick up a change when they refresh this marketplace.

## Install

Claude Code:

```sh
claude plugin marketplace add ryangerardwilson/skills
claude plugin install r2d2-skills@skills
```

Grok Build:

```sh
grok plugin marketplace add ryangerardwilson/skills
grok plugin install ryangerardwilson/skills#plugins/r2d2-skills --trust
```

Grok keeps a plugin off until it is enabled. Add `r2d2-skills` to
`[plugins] enabled` in `~/.grok/config.toml`, or run
`grok plugin enable r2d2-skills`.

Codex:

```sh
codex plugin marketplace add ryangerardwilson/skills
codex plugin add r2d2-skills@skills
```

## Update

```sh
claude plugin marketplace update skills && claude plugin update r2d2-skills@skills
grok plugin marketplace update skills && grok plugin update r2d2-skills
codex plugin marketplace upgrade skills && codex plugin add r2d2-skills@skills
```

Codex copies the plugin into its cache, so `codex plugin add` runs again
after the upgrade to refresh that copy.

## Layout

```
.claude-plugin/marketplace.json     Claude Code marketplace
.grok-plugin/marketplace.json       Grok Build marketplace
.agents/plugins/marketplace.json    Codex marketplace
plugins/r2d2-skills/.claude-plugin/plugin.json
plugins/r2d2-skills/.grok-plugin/plugin.json
plugins/r2d2-skills/.codex-plugin/plugin.json
plugins/r2d2-skills/skills/<name>/SKILL.md
```

A `SKILL.md` is saved exactly as it was written. The directory name is the
skill name. The file does not need a `name` or `description` header.
