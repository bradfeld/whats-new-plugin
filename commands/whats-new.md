---
description: "Cross-reference Claude Code release notes against your config"
effort: medium
---

# What's New

## Help Mode

If the argument is `?` or `--help`, print this block and STOP — do not execute any steps:

```
Cross-reference Claude Code release notes against your personal config.
Surfaces what changed that affects your setup, recommends new features
worth adopting, and summarizes the rest.

Usage: /whats-new [version]

Arguments:
  (none)          Analyze all releases since last reviewed
  <version>       Analyze a specific version only (e.g., 2.1.83)
                  Does not update the last-reviewed tracking.

Examples:
  /whats-new                  Full analysis since last review
  /whats-new 2.1.83           Analyze a specific version
  /whats-new ?                Show this help
```

---

## Step 1 — Parse Arguments

Read `$ARGUMENTS`:
- If empty: set `mode = since-last-review`
- If the value matches a semver pattern like `2.1.83` (digits, dot, digits, dot, digits): set `mode = specific-version`, set `target_version` = that value
- If `?` or `--help`: already handled by Help Mode above — stop here

## Step 2 — Get Current Version

Use Bash tool to run:

```bash
claude --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+'
```

Store the result as `current_version`.

## Step 3 — Determine Version Range

**If `mode = since-last-review`:**

1. Use the Read tool to read `~/.claude/whats-new-last-version.txt`
2. If the file exists and has content: set `from_version` = file contents (trimmed of whitespace)
3. If the file is missing or empty: skip to Step 4 to fetch the releases list first, then:
   - Find `current_version` in the newest-first release list
   - Set `from_version` = the version 5 positions further into the list (i.e., `releases[current_idx + 5]`)
   - If `current_version` is not in the list, use the most recent published release as `from_version`

Set `to_version` = `current_version`.

**If `mode = specific-version`:**

1. Skip to Step 4 to fetch the releases list first, then:
   - Find `target_version` in the newest-first release list at index `target_idx`
   - Set `from_version` = the version at index `target_idx + 1` (one slot further into the past)
   - Set `to_version` = `target_version`

**Guard clause:** If `from_version` equals `to_version`, or the releases list filtered to the range is empty, output:

> You're up to date — no new releases since v{current_version}.

Then STOP.

## Step 4 — Fetch Release Notes

Use Bash tool:

```bash
curl -s "https://api.github.com/repos/anthropics/claude-code/releases?per_page=100"
```

Parse the JSON response:
- For each release, strip the `v` prefix from `tag_name` for version comparison
- Collect releases where version > `from_version` AND version <= `to_version`
- Extract the `body` field (markdown release notes) for each matching release
- Keep the newest-first ordering

On API failure (non-200, malformed JSON, empty response): report the error clearly and suggest trying again later. Do not proceed.

## Step 5 — Build Config Inventory

Read and summarize as structured plain text. Do not dump raw file contents.

**5a. Hooks (from settings.json)** — Use Read tool on `~/.claude/settings.json`. For each key in the `hooks` object, record the lifecycle event name and the script filenames extracted from each hook's `command` field.

**5b. Hook files on disk** — Use Glob to list `~/.claude/hooks/*.sh`. List the filenames. This catches scripts that exist on disk but are not registered in settings.json.

**5c. Env vars** — From the `env` object in `settings.json`, list the key names only (not values).

**5d. Rules** — Glob `~/.claude/rules/*.md` for global rules. Glob `{current_project}/.claude/rules/*.md` for project-level rules (use the working directory). List filenames without extensions.

**5e. Skills** — List top-level directory names under `~/.claude/skills/`.

**5f. Commands** — Glob `~/.claude/commands/*.md` for global commands. Glob `{current_project}/.claude/commands/*.md` for project-level commands. List filenames without extensions.

**5g. Plugins** — From the `enabledPlugins` array in `settings.json`, list each plugin name and whether it is enabled.

**5h. Other config** — From `settings.json`, extract these fields if present: `outputStyle`, `statusLine`, `permissions` (count patterns), `alwaysThinkingEnabled`, `autoUpdatesChannel`.

Format the inventory as:

```
Hooks (N scripts across M lifecycle events):
  SessionStart: script1.sh, script2.sh
  PreToolUse: script3.sh
  ...
Hook files on disk (N): script1.sh, script2.sh, ...

Env vars (N): VAR_ONE, VAR_TWO, ...

Rules (N, global): rule-name-1, rule-name-2, ...
Rules (N, project): rule-name-a, ...

Skills (N): skill-one, skill-two, ...

Commands (N, global): command-one, command-two, ...
Commands (N, project): command-a, ...

Plugins (N): superpowers (enabled), simmer (enabled), ...

Other: outputStyle=explanatory, statusLine=command, permissions=12 patterns, ...
```

## Step 6 — Cross-Reference and Analyze

For each item in the collected release notes, classify it into exactly one category:

1. **IMPACT** — The change directly affects an existing config element (a specific hook, rule, command, skill, plugin, env var, or setting that appears in the inventory). Name the specific element(s) and describe what to check or update.

2. **RECOMMENDATION** — The change introduces a new capability that has a concrete intersection with the user's config setup — e.g., a new hook lifecycle event, a new settings.json field, a new built-in behavior that a custom command or rule already approximates. Only classify as RECOMMENDATION when the intersection is real and specific. Not every new feature is a recommendation.

3. **GENERAL** — Everything else. Summarize in one line.

## Step 7 — Output the Report

Print the following three sections in order.

---

### Impacts Your Config

List each IMPACT item:

**v{version}: {release note item title or summary}**
→ Affects: {specific config element name(s)}
→ Action: {what to check, update, or verify}

If there are no IMPACT items, print: *(No changes impact your current config.)*

---

### Recommendations

Omit this section entirely if there are no RECOMMENDATION items.

If present, list each:

**v{version}: {capability}**
→ Connects to: {specific config element}
→ Suggestion: {concrete, actionable suggestion — what to add, change, or consider}

---

### General

One bullet per GENERAL item, grouped under a `**v{version}**` subheading. If 20 or more versions are covered, use strict one-liners (no sub-bullets, no elaboration).

---

## Step 8 — Update Tracking

**Only execute this step if `mode = since-last-review`.** Do NOT run for `specific-version` mode.

Use Bash tool to write the current version to the tracking file:

```bash
printf '%s' '{current_version}' > ~/.claude/whats-new-last-version.txt
```

Replace `{current_version}` with the actual version string. Use `printf '%s'` (not `echo`) to avoid a trailing newline.

After writing, report:

> Updated last-reviewed version to v{current_version}.
