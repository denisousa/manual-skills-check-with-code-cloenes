<!-- tsim82/internal-ai-marketing-team:uploads/extracted/cleanup/SKILL.md | https://github.com/tsim82/internal-ai-marketing-team/blob/HEAD/uploads/extracted/cleanup/SKILL.md -->

---
name: cleanup
description: Weekly workspace audit - finds bloat, stale files, broken references, and organizational drift across all projects
---

<objective>
You are a workspace hygiene agent. Audit every project folder in ~/your-workspace/ and report what needs cleaning. Be opinionated - flag problems, propose fixes, and wait for approval before changing anything.

Run this weekly (Sunday or Monday morning) to catch drift before it compounds.
</objective>

<instructions>

## Phase 1: Scan (Do This Silently)

For each project folder in `~/your-workspace/` (and nested subfolders):

### 1.1 CLAUDE.md Health
- Read each CLAUDE.md and count lines
- Flag if over **80 lines** (behavioral instructions should be lean; reference data belongs in `reference/`)
- Check the reference file table - does every listed file actually exist? Flag broken references.

### 1.2 TASKS.md Health
- Check if a TASKS.md (or equivalent task file) exists
- Flag if over **100 lines** (tasks should be one-line checkboxes, not verbose paragraphs)
- Count completed vs incomplete tasks
- Flag if >80% tasks are complete (time to archive and reset)

### 1.3 Root Clutter
- List all files at the project root (not in subfolders)
- Flag files that look like reference data at root level:
  - `.txt` files (usually raw docs that belong in `reference/`)
  - Files over 200 lines at root (probably reference material)
  - Files with names like `*-spec.*`, `*-schema.*`, `*-config.*`, `*-system.*`
  - Avatar, offer, framework, outline files
- The only files that should live at root are: `CLAUDE.md`, `TASKS.md` (or `ROADMAP.md`), and folders

### 1.4 Empty Folders
- Find directories with no files in them
- Flag for deletion

### 1.5 Stale Content
- Check for files not modified in 60+ days that aren't reference material
- Check for dated files (like `analysis-2025-*.md`) older than 90 days
- Check `reference/` for placeholder files (files under 5 lines that say "placeholder" or "TBD")

### 1.6 Duplicate Detection
- Check if any CLAUDE.md contains n8n build patterns (these belong ONLY in `~/.claude/CLAUDE.md`)
- Check if workflow IDs, table IDs, or API credentials appear in more than one file
- Check if content from `reference/` files is duplicated in CLAUDE.md

### 1.7 Cross-File References
- Read `~/.claude/commands/morning.md` and check that all file paths it references still exist
- Read each CLAUDE.md and check that all referenced files/folders exist

## Phase 2: Report

Output a structured report:

```markdown
# Weekly Cleanup Report - [Date]

## Summary
- Projects scanned: [N]
- Issues found: [N]
- Quick fixes (auto): [N]
- Needs review: [N]

---

## Issues by Project

### [Project Name]

#### [severity emoji] [Issue Title]
**What:** [Description]
**Fix:** [Proposed action]
**Impact:** [Why this matters]

---
```

Severity levels:
- 🔴 **Bloat** - CLAUDE.md over 80 lines, task file over 100 lines (wastes context every session)
- 🟡 **Drift** - files at wrong location, broken references, stale content
- 🟢 **Hygiene** - empty folders, naming inconsistencies, placeholder files

## Phase 3: Fix

After showing the report, ask:

> **Which fixes should I apply?** (all / pick by number / none)

Then execute approved fixes:
- Move files to `reference/` with proper names
- Delete empty folders
- Update CLAUDE.md reference tables
- Remove duplicate content
- Fix broken file paths in commands

**Rules:**
- NEVER delete content files without asking - only move them
- NEVER modify TASKS.md task status - only flag for user review
- ALWAYS update CLAUDE.md reference tables after moving files
- ALWAYS show what changed after fixes are applied

## Phase 4: Summary

After fixes, output:

```markdown
## Changes Applied
- [list of changes]

## Still Needs Manual Attention
- [anything you couldn't auto-fix]

**Next cleanup:** [date + 7 days]
```

</instructions>

<thresholds>

These are the organizational rules. Flag anything that violates them:

| Rule | Threshold | Why |
|------|-----------|-----|
| CLAUDE.md max lines | 80 | Everything above is reference data wasting context |
| TASKS.md max lines | 100 | Tasks should be one-line checkboxes |
| Root files allowed | CLAUDE.md, TASKS.md, ROADMAP.md, folders only | Everything else goes in reference/ |
| Empty folders | 0 tolerance | Delete them |
| Placeholder files | Flag if still placeholder after 2+ weeks | Fill them or delete them |
| Duplicate n8n rules | Only in ~/.claude/CLAUDE.md | Never in project CLAUDE.md files |
| Broken references | 0 tolerance | Fix immediately |

</thresholds>
