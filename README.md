# whats-new

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that cross-references release notes against your personal config to surface what actually matters to you.

Instead of reading through every changelog entry, `/whats-new` scans your `~/.claude/settings.json`, hooks, rules, skills, commands, and plugins, then classifies each release note into:

- **Impacts Your Config** -- changes that directly affect something you have configured
- **Recommendations** -- new capabilities with a concrete intersection with your setup
- **General** -- everything else, summarized in one line

## Install

**Step 1: Add the marketplace**

```bash
claude plugins marketplace add https://github.com/bradfeld/whats-new-plugin.git
```

**Step 2: Install the plugin**

```bash
claude plugins install whats-new
```

Then restart Claude Code to load the new command.

## Usage

```
/whats-new                  Analyze all releases since last reviewed
/whats-new 2.1.83           Analyze a specific version
/whats-new ?                Show help
```

### Since Last Review (default)

Running `/whats-new` with no arguments checks all releases since the last time you ran it. It tracks the last-reviewed version in `~/.claude/whats-new-last-version.txt`.

On first run (no tracking file), it defaults to the last 5 releases.

### Specific Version

Running `/whats-new 2.1.83` analyzes just that version. This mode does not update the tracking file.

## What It Scans

The plugin inventories your Claude Code configuration:

| Config Type | Source |
|-------------|--------|
| Hooks | `settings.json` hooks + `~/.claude/hooks/*.sh` on disk |
| Env vars | `settings.json` env keys |
| Rules | `~/.claude/rules/*.md` + project `.claude/rules/*.md` |
| Skills | `~/.claude/skills/` directories |
| Commands | `~/.claude/commands/*.md` + project `.claude/commands/*.md` |
| Plugins | `settings.json` enabledPlugins |
| Other settings | outputStyle, statusLine, permissions, etc. |

It then cross-references this inventory against release notes fetched from the [Claude Code GitHub releases](https://github.com/anthropics/claude-code/releases).

## License

MIT
