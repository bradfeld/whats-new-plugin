# whats-new

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that cross-references release notes against your personal config to surface what actually matters to you.

Instead of reading through every changelog entry, `/whats-new` scans your `~/.claude/settings.json`, hooks, rules, skills, commands, and plugins, then classifies each release note into:

- **Impacts Your Config** -- changes that directly affect something you have configured
- **Recommendations** -- new capabilities with a concrete intersection with your setup
- **General** -- everything else, summarized in one line

Each potential impact is then **verified with concrete tool calls** (grep, file checks, platform detection) before it lands in the final report. Items that the verification proves don't actually affect your config are demoted to General — so the "Impacts Your Config" section only contains things you've confirmed apply to your setup.

## Install

**Step 1: Add the marketplace**

```bash
claude plugins marketplace add https://github.com/bradfeld/whats-new-plugin.git
```

**Step 2: Install the plugin**

```bash
claude plugins install release-radar
```

Then restart Claude Code to load the new `/whats-new` command.

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

### Evolve Config

In the default (since-last-review) mode, after the report `/whats-new` turns the actionable findings into concrete config edits. **Every candidate is independently fresh-eyes-reviewed before you see it** — a separate reviewer vets each proposed edit (revising or flagging it), and the verdict is folded into the list. It then presents the vetted changes as a numbered list — each with the literal change, a plain-language explanation, and the review's verdict (clean / revised / ⚠ recommends-against) — and waits:

```
Reply with numbers to apply ("1 3"), "all", or "none".
```

Nothing is applied without your explicit pick. Every applied change is validated (`jq` / `bash -n`) and logged to `~/.claude/whats-new-applied.md` — a cumulative record of how your config has evolved release-over-release. Specific-version runs are inspect-only and never modify config.

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
