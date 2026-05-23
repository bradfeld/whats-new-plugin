---
description: "Cross-reference Claude Code release notes against your config"
effort: medium
---

# What's New

## Help Mode

If the argument is `?` or `--help`, print this block and STOP — do not execute any steps:

```
Cross-reference Claude Code release notes against your personal config,
then verify each impact item with concrete tool calls before reporting.
Surfaces what changed that actually requires action, recommends new
features worth adopting, and summarizes the rest.

Usage: /whats-new [version]

Arguments:
  (none)          Analyze all releases since last reviewed
  <version>       Analyze a specific version only (e.g., 2.1.83)
                  Does not update the last-reviewed tracking.

Examples:
  /whats-new                  Full analysis since last review
  /whats-new 2.1.83           Analyze a specific version
  /whats-new ?                Show this help

Each IMPACT item is verified with grep / read / file-check before being
reported as actionable. Items that the verification proves don't affect
your config are demoted to GENERAL — no false alarms.
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

**5b. Hook and helper scripts on disk** — Use Glob to list `~/.claude/hooks/*.sh` and `~/.claude/scripts/*.sh`. List the filenames from each path separately. This catches scripts that exist on disk but are not registered in settings.json, including ad-hoc helpers under `scripts/` that hooks call into.

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

## Step 6.5 — Verify each IMPACT before reporting

For every item classified as IMPACT in Step 6, run a verification pass before final reporting. The goal is to convert "this might affect your config" into a concrete, evidence-backed claim before it lands in the report.

### 6.5a — Derive a verification recipe

Match the release-note text against the table below. If no row matches, generate an inline recipe based on the inventory from Step 5 and the specific config element(s) named in the IMPACT.

| Release-note pattern | Verification recipe |
|---|---|
| "sandbox" + "worktree" / "main repo" / "shared `.git`" | Run `git worktree list` to enumerate worktrees. `grep -rln "<main-repo-name>" ~/.claude/hooks/ ~/.claude/scripts/` and inspect for writes targeting the main repo root or its `.git/{hooks,config}` from worktree contexts. Absence of such writes = verified safe. |
| "otelHeadersHelper" / "otel.*helper" | `grep -rE "otelHeadersHelper" ~/.claude/settings.json ~/.claude/hooks/ ~/.claude/scripts/`. No match = N/A; demote to GENERAL. |
| "Bash tool" + path/permission/cd | Extract the specific pattern from the release note (a path, a flag, a command). Substitute into `grep -rE "<pattern>" ~/.claude/hooks/ ~/.claude/scripts/`. Cross-check `permissions.allow` / `permissions.deny` in settings.json for matching rules. |
| "PowerShell" / "Windows" / "`.exe`" / "Windows-only" | Platform check: `uname` — if Darwin or Linux, mark N/A and demote to GENERAL. |
| "managed setting" / "Enterprise" / "managed-mcp.json" | Check for managed-settings markers: `ls /Library/Application\ Support/ClaudeCode/ 2>/dev/null`; absent = N/A. |
| "skill frontmatter" / "agent frontmatter" / "`effort:`" / "`name:`" | Glob the inventory's skills/agents lists; grep for the affected frontmatter field. Confirm at least one file uses it before claiming impact. |
| "/<command>" referencing a builtin slash command | Check for user override: `ls ~/.claude/commands/<command>.md 2>/dev/null`. Override present → IMPACT-ACTIONABLE (review for conflict); absent → typically GENERAL. |
| "MCP" + specific server name | Check `enabledPlugins` + any `.mcp.json` files for that server. Absent = N/A. |
| "hook" + lifecycle event | Grep the `hooks` block of settings.json for that event name. Empty list for that event = N/A. |
| "permission" + pattern | Grep `permissions.allow` and `permissions.deny` for the pattern. |
| "remote session" / "Remote Control" / "claude.ai" | Check whether user has remote-session usage signals (e.g., `~/.claude/remote-sessions/` or similar). Absent = N/A. |
| "`/<command>`" output formatting / display fix | Pure UI fix — verify by reading the inventory's `outputStyle` and `statusLine` only; almost always N/A. |
| (no row matches) | Generate inline: name the specific config element from the IMPACT, choose the cheapest concrete check (Read / Glob / Grep / Bash), execute it, and capture findings. |

### 6.5b — Execute the recipe

Run the recipe using Bash / Read / Glob / Grep. Capture concrete evidence: file paths, line numbers, counts, presence/absence. Do not skip this step — assertions without evidence are exactly what this stage exists to eliminate.

If a recipe requires writing (it shouldn't — verification is read-only), STOP and flag the recipe as malformed.

### 6.5c — Reclassify based on findings

| Finding | New classification | How to report |
|---|---|---|
| Verification surfaces a real exposure, conflict, or required change | IMPACT-ACTIONABLE | Keep in IMPACT section with specific action steps |
| Fix applies to user's setup but no action needed (security tightening, behavior improvement) | IMPACT-VERIFIED-SAFE | Keep in IMPACT section with "Verified safe: <reason>" |
| Verification proves the change does not affect this config | IMPACT-NOT-APPLICABLE (i.e., demoted to GENERAL) | Move to General section, mention briefly in one line |

Record the recipe summary and the finding alongside each IMPACT item — they appear in the final report.

### 6.5d — Parallel verification

When 2+ IMPACT items have independent verification recipes (touching different config surfaces), bundle their Bash/Grep calls in a single message so they run in parallel. There is no count threshold below which parallelization is skipped — even 2 verifications run faster as one batch than two sequential round-trips. Do NOT skip verification for any IMPACT to save time — false alarms in this report are more expensive than the extra tool calls.

## Step 7 — Output the Report

Print the following three sections in order.

---

### Impacts Your Config

List each IMPACT-ACTIONABLE and IMPACT-VERIFIED-SAFE item (demoted items go to General):

**v{version}: {release note item title or summary}**
→ Affects: {specific config element name(s)}
→ Verified: {one-line recipe summary} — {finding}
→ Action: {actionable steps OR "None needed — verified safe (<reason>)"}

If there are no IMPACT items after verification, print: *(No changes impact your current config.)*

---

### Recommendations

Omit this section entirely if there are no RECOMMENDATION items.

If present, list each:

**v{version}: {capability}**
→ Connects to: {specific config element}
→ Suggestion: {concrete, actionable suggestion — what to add, change, or consider}

---

### General

One bullet per GENERAL item, grouped under a `**v{version}**` subheading. **Include items demoted from IMPACT-NOT-APPLICABLE in Step 6.5** — they aren't dropped; they move here. If 20 or more versions are covered, use strict one-liners (no sub-bullets, no elaboration).

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
