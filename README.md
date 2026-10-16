# work-share

Skills and config shared between machines.

## Skills

Canonical copies live in the private `agent-conf` repo; these are synced copies.

| Skill | Command | What it does |
|---|---|---|
| [guideme](skills/guideme/SKILL.md) | `/guideme` | Plans and walks you through a setup Claude can't do directly, keeping a LOG + IMP in the Obsidian vault and a local mirror. |

### Install

```sh
git clone https://github.com/dmtrio/work-share.git
mkdir -p ~/.claude/skills
cp -R work-share/skills/guideme ~/.claude/skills/
```

Start a new Claude Code session and `/guideme` is available.

`guideme` expects an Obsidian MCP server named `obsidian` (for the vault copy, under `GUIDES/`) and web search/fetch for research. Without the Obsidian server it still writes the local mirror under `~/.claude/guideme/`.
