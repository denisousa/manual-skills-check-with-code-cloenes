<!-- mnismt/changelogs.directory:.agents/skills/updating-docs/SKILL.md | https://github.com/mnismt/changelogs.directory/blob/HEAD/.agents/skills/updating-docs/SKILL.md -->

---
name: updating-docs
description: Updates project documentation to match code changes. Use after implementing features to sync docs with code, when asked to update/fix documentation, or to mark tasks complete in TASKS.md.
---

# Updating Docs Skill

Keeps documentation in sync with code changes. Validates, updates, creates, and suggests deletion of docs. **Automatically updates TASKS.md** when code changes indicate task completion.

## Constraints

**DO NOT:**
- Modify source code files—only documentation
- Assume what code does—always read it first
- Delete docs without user confirmation
- Skip reading relevant code before updating docs

**DO:**
- Read code to understand current implementation
- Update docs to reflect actual state
- Create new docs for new features
- Suggest deleting obsolete docs
- Update README.md navigation when adding/removing docs
- **Mark tasks complete in TASKS.md** when code changes indicate completion
- Validate existing docs for accuracy
- Include TanStack Start patterns where relevant

## Workflow

### Phase 1: Identify What Changed

**Option A: Auto-detect via git**
```bash
# Recent file changes
git diff --name-only HEAD~5

# Or specific commit range
git diff --name-only <commit1>..<commit2>

# With context
git log --oneline -10
```

**Option B: User specifies**
Ask: "What code did you change or implement?"

**Option C: Full validation**
If user asks to validate all docs, scan entire codebase against docs.

### Phase 2: Map Code to Docs

Use the mapping in `code-doc-mapping.md` to identify affected docs:

| Code Location | Update Doc |
|---------------|------------|
| `src/routes/*` | `docs/reference/api-patterns.md` |
| `src/server/*` | `docs/reference/api-patterns.md` |
| `src/trigger/ingest/*` | `docs/reference/ingestion-pipeline.md` |
| `src/trigger/ingest/<tool>/parser.ts` | `docs/reference/parsers.md` |
| `src/components/*` | `docs/design/design-rules.md` |
| `prisma/schema.prisma` | `docs/reference/database-schema.md` |
| `prisma/seed.ts` | `docs/guides/adding-a-tool.md` |

For complete mapping, see `code-doc-mapping.md`.

### Phase 3: Validate & Update Each Doc

For each affected doc:

1. **Read the current doc** to understand what it claims
2. **Read the actual code** to see current implementation
3. **Compare and identify discrepancies**:
   - Missing features/APIs
   - Outdated function signatures
   - Incorrect descriptions
   - Stale examples
   - Missing TanStack Start patterns
4. **Update doc** to match current implementation
5. **Add TanStack patterns** where relevant (SSR, server functions)

### Phase 4: Handle New Docs

If a new feature area was added without docs:

1. **Identify the correct directory** based on doc structure:
   - Task-oriented guides → `docs/guides/`
   - Technical deep-dives → `docs/reference/`
   - UI/UX guidelines → `docs/design/`
   - Project planning/status → `docs/project/`

2. **Create doc following conventions**:
   - Use kebab-case filename
   - Include header with title
   - Add to README.md navigation

3. **Update docs/README.md**:
   - Add to appropriate section
   - Update quick reference if applicable

### Phase 5: Auto-Update TASKS.md

When code changes indicate a task is complete:

1. **Read `docs/project/TASKS.md`** to find matching task
2. **Verify the implementation matches the task** description
3. **Change `- [ ]` to `- [x]`** for completed items
4. **Add brief completion note** if helpful (optional)

**Examples of task completion triggers:**
| Code Change | Task to Mark Complete |
|-------------|----------------------|
| Added `src/trigger/ingest/cursor/` | "Add Cursor changelog ingestion" |
| Created `src/routes/tools/$slug/index.tsx` | "Implement tool detail page" |
| Added parser tests | "Add parser tests" |

### Phase 6: Suggest Deletions

Identify obsolete docs when:
- Code they document was removed
- Feature was deprecated/replaced
- Doc duplicates another doc

**Output format for deletions:**
```
## Suggested Deletions

| Doc | Reason | Action |
|-----|--------|--------|
| `docs/path/file.md` | Feature X was removed in commit abc123 | Delete or archive? |
```

Wait for user confirmation before deleting.

### Phase 7: Final Verification

- [ ] All affected docs updated
- [ ] No contradictions between docs
- [ ] README.md reflects current doc structure
- [ ] Internal links still work (check `[text](path)` references)
- [ ] TASKS.md updated for completed work
- [ ] TanStack Start patterns documented correctly

## TanStack Start Patterns to Document

When updating docs, ensure these critical patterns are included:

### SSR Data Fetching
```tsx
// ✅ Document this pattern
export const Route = createFileRoute('/path/')({
  loader: async () => await serverFunction(),
  component: PageComponent,
})

function PageComponent() {
  const data = Route.useLoaderData()
}
```

### Database Access
```tsx
// ✅ Document this pattern
// In src/server/
export const getData = createServerFn({ method: 'GET' }).handler(async () => {
  const prisma = getPrisma()
  return prisma.model.findMany()
})

// ❌ Document what NOT to do
// NEVER import getPrisma in routes
```

### File-Based Routing
Document the routing conventions:
- `index.tsx` → `/path`
- `$param.tsx` → `/path/:param`
- `__root.tsx` → Layout wrapper

## Doc Conventions

When writing/updating docs:

1. **Keep concise** - No fluff, only useful info
2. **Use tables** for structured data (APIs, mappings, etc.)
3. **Code examples** should be real, working code
4. **Cross-reference** other docs rather than duplicating
5. **DRY principle** - Deep docs are authoritative; indexes only link
6. **Include critical patterns** - SSR, server functions, routing

## Quick Reference: Doc Structure

```
docs/
├── README.md                # Entry point, navigation
├── guides/                  # Task-oriented how-tos
│   ├── adding-a-tool.md     # 6-step guide for new tools
│   ├── deployment.md        # Production deployment
│   ├── environment-variables.md
│   └── testing.md
├── reference/               # Technical deep-dives
│   ├── api-patterns.md      # SSR, server functions
│   ├── database-schema.md   # Prisma models
│   ├── ingestion-pipeline.md # 7-phase pipeline
│   └── parsers.md           # Parser development
├── design/                  # UI/UX guidelines
│   ├── design-rules.md      # Core aesthetic (MUST READ for UI)
│   ├── og-images.md
│   └── animations/
└── project/                 # Planning & status
    ├── architecture.md
    ├── prd.md
    └── TASKS.md             # MVP task tracking
```

## TASKS.md Update Examples

### Before completing a task:
```markdown
- [ ] Add Cursor changelog parser
```

### After completing:
```markdown
- [x] Add Cursor changelog parser
```

### With completion note (optional):
```markdown
- [x] Add Cursor changelog parser (completed: src/trigger/ingest/cursor/)
```

## When to Use This Skill

- "Update the docs to match what I just implemented"
- "Sync documentation with code changes"
- "Validate the docs are accurate"
- "Check if docs are up to date"
- "Document the new feature I added"
- "Clean up outdated documentation"
- "Mark this task as complete"
- "Update TASKS.md"
- "What docs need updating after these changes?"
