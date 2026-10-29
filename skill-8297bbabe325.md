<!-- Thedougler/shattered-sea-wiki:.claude/skills/obsidian-bases/SKILL.md | https://github.com/Thedougler/shattered-sea-wiki/blob/HEAD/.claude/skills/obsidian-bases/SKILL.md -->

---
name: obsidian-bases
description: >
  Create and edit Obsidian Bases (.base files): Obsidian's native database layer for
  dynamic tables, card views, list views, filters, formulas, and summaries over vault notes.
  Canonical triggers: "create a base", "obsidian bases", "database view".
  Full trigger list in the skill body.
---

# Obsidian Bases: Obsidian's Database Layer

## When to use

Full trigger set: "create a base", "obsidian bases", "base view", "filter notes", "formula", "database view", "dynamic table", "dashboard base".

Obsidian Bases (launched 2025) turns vault notes into queryable, dynamic views. Tables,
cards, lists. Defined in `.base` files. No plugin required — it is a core Obsidian feature
(requires Obsidian v1.9.10+).

Official docs: https://help.obsidian.md/bases/syntax

---

## File Format

`.base` files contain valid YAML. The root keys are `filters`, `formulas`,
`properties`, `summaries`, and `views`.

```yaml
# Global filters: apply to ALL views
filters:
  and:
    - file.hasTag("wiki")
    - 'status != "archived"'

# Computed properties
formulas:
  age_days: '(now() - file.ctime).days.round(0)'
  status_icon: 'if(status == "mature", "✅", "🔄")'

# Display name overrides for properties panel
properties:
  status:
    displayName: "Status"
  formula.age_days:
    displayName: "Age (days)"

# One or more views
views:
  - type: table
    name: "All Pages"
    order:
      - file.name
      - type
      - status
      - updated
      - formula.age_days
```

---

## Filters

Filters select which notes appear. Applied globally or per-view.

```yaml
# Single string filter
filters: 'status == "current"'

# AND: all must be true
filters:
  and:
    - 'status != "archived"'
    - file.hasTag("wiki")

# OR: any can be true
filters:
  or:
    - file.hasTag("concept")
    - file.hasTag("entity")

# NOT: exclude matches
filters:
  not:
    - file.inFolder("wiki/meta")

# Nested
filters:
  and:
    - file.inFolder("wiki/")
    - or:
        - 'type == "concept"'
        - 'type == "entity"'
```

### Filter operators

`==` `!=` `>` `<` `>=` `<=`

### Useful filter functions

| Function | Example |
|----------|---------|
| `file.hasTag("x")` | Notes with tag `x` |
| `file.inFolder("path/")` | Notes in folder |
| `file.hasLink("Note")` | Notes linking to Note |

---

## Properties

Three types:
- **Note properties**: from frontmatter — `status`, `type`, `updated`
- **File properties**: metadata — `file.name`, `file.mtime`, `file.size`, `file.ctime`, `file.tags`, `file.folder`
- **Formula properties**: computed — `formula.age_days`

---

## Formulas

Defined in `formulas:`. Referenced as `formula.name` in `order:` and `properties:`.

```yaml
formulas:
  # Days since created
  age_days: '(now() - file.ctime).days.round(0)'

  # Conditional label
  status_icon: 'if(status == "active", "✅", if(status == "deceased", "💀", if(status == "unmet", "👁️", "🔄")))'

  # Word count estimate
  word_est: '(file.size / 5).round(0)'
```

**Key rule**: Subtracting two dates returns a `Duration`. Not a number. Always access `.days` first:
```yaml
# CORRECT
age: '(now() - file.ctime).days'

# WRONG: crashes
age: '(now() - file.ctime).round(0)'
```

**Always guard nullable properties with `if()`**:
```yaml
days_left: 'if(due_date, (date(due_date) - today()).days, "")'
```

---

## View Types

### Table
```yaml
views:
  - type: table
    name: "Wiki Index"
    limit: 100
    order:
      - file.name
      - type
      - status
      - updated
      - formula.age_days
    groupBy:
      property: type
      direction: ASC
```

### Cards
```yaml
views:
  - type: cards
    name: "Gallery"
    order:
      - file.name
      - tags
      - status
```

### List
```yaml
views:
  - type: list
    name: "Quick List"
    order:
      - file.name
      - status
```

---

## Shattered Sea Wiki Templates

### Full wiki dashboard (all non-meta pages)

```yaml
filters:
  and:
    - file.inFolder("wiki/")
    - not:
        - file.inFolder("wiki/meta")

formulas:
  age: '(now() - file.ctime).days.round(0)'

properties:
  formula.age:
    displayName: "Age (days)"

views:
  - type: table
    name: "All Wiki Pages"
    order:
      - file.name
      - type
      - status
      - updated
      - formula.age
    groupBy:
      property: type
      direction: ASC
```

### NPC tracker

```yaml
filters:
  and:
    - file.inFolder("wiki/entities/npcs/")
    - 'file.ext == "md"'

formulas:
  confirmations: 'if(sources, length(sources), 0)'

views:
  - type: table
    name: "NPCs"
    order:
      - file.name
      - status
      - type
      - formula.confirmations
      - updated
    groupBy:
      property: status
      direction: ASC
```

### Entity index (all entity types)

```yaml
filters:
  and:
    - 'file.ext == "md"'
    - 'type != "source" && type != "meta" && type != "domain-index"'

views:
  - type: table
    name: "Entities"
    order:
      - file.name
      - type
      - status
      - updated
    groupBy:
      property: type
      direction: ASC
```

### Source tracker

```yaml
filters:
  and:
    - file.inFolder("wiki/reference/")

views:
  - type: table
    name: "Sources"
    order:
      - file.name
      - created
      - status
    groupBy:
      property: status
      direction: ASC
```

---

## Embedding in Notes

```markdown
![[MyBase.base]]

![[MyBase.base#View Name]]
```

---

## When to Use Bases

Bases are **canonical infrastructure**, not optional visualizations. They are the primary
mechanism for keeping overview and aggregation pages DRY (see CLAUDE.md § DRY & dynamic
content).

### Decision Tree

| Content type | Use | Example |
|---|---|---|
| List/roster/table of entities with status, counts, or other frontmatter fields | **Base embed** | `![[party-roster.base]]`, `![[faction-tracker.base]]` |
| A specific entity's summary, lore excerpt, or read-aloud block shared across pages | **Section embed** (`![[Entity#Section]]`) | `![[Pearl-of-Souls#Lore]]` |
| Cross-reference where the reader navigates to the full page | **Wikilink** (`[[Entity]]`) | `[[The-Dravosi-Crown]]` |
| Content unique to this page, not derivable from entity frontmatter or sections | **Static text** | GM-only commentary, cross-entity synthesis |
| A hand-written table listing entity names + their status/active problem/location | **Replace with base embed** — this is the anti-pattern | Static table → `![[tracker.base]]` |

### When creating or updating overview/aggregation pages

1. **Always check** if a `.base` file in `wiki/meta/` already covers the data
2. **If a base exists:** embed it instead of writing a static table
3. **If no base exists and the data is frontmatter-queryable:** create a `.base` file
   in `wiki/meta/`, then embed it
4. **If the data is NOT frontmatter-queryable** (e.g., narrative synthesis): static text
   is acceptable, but keep it compressed to 1–2 sentences with wikilinks

### Bases query frontmatter only

Bases **cannot read markdown body text**. They can only query:
- YAML frontmatter properties (`status`, `type`, `active_problem`, `faction`, etc.)
- File metadata (`file.name`, `file.mtime`, `file.ctime`, `file.size`, `file.tags`)
- Computed formulas from the above

This means fields that need to appear in base views **must exist in frontmatter**.
All toy fields (including `active_problem`) live in YAML frontmatter only.
See CLAUDE.md § DRY & dynamic content.

---

## Where to Save

Store `.base` files in `wiki/meta/` for vault dashboards:
- `wiki/meta/dashboard.base`: main content view
- `wiki/meta/entities.base`: entity tracker
- `wiki/meta/npcs.base`: NPC tracker
- `wiki/meta/sources.base`: source log

---

## YAML Quoting Rules

- Formulas with double quotes → wrap in single quotes: `'if(done, "Yes", "No")'`
- Strings with colons or special chars → wrap in double quotes: `"Status: Active"`
- Unquoted strings with `:` break YAML parsing

---

## What NOT to Do

- Do not use `from:` or `where:` — those are Dataview syntax, not Obsidian Bases
- Do not use `sort:` at the root level — sorting is per-view via `order:` and `groupBy:`
- Do not put `.base` files outside the vault — they only render inside Obsidian
- Do not reference `formula.X` in `order:` without defining `X` in `formulas:`
