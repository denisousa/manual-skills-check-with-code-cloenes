<!-- fyrsmithlabs/marketplace:plugins/fs-dev/skills/roadmap-discovery/SKILL.md | https://github.com/fyrsmithlabs/marketplace/blob/HEAD/plugins/fs-dev/skills/roadmap-discovery/SKILL.md -->

---
name: roadmap-discovery
description: Use when analyzing codebase for improvements, running discovery, finding issues, auditing security/quality/performance/compliance/accessibility/DX, identifying tech debt, or scanning for problems. Autonomous non-interactive analysis with lens filtering (security, quality, perf, docs, dx, compliance, a11y) and severity classification (CRITICAL, MAJOR, MINOR, SUGGESTION). Supports parallel lens execution, complexity scoring, trend analysis, and multi-format output (GitHub Issues, JIRA, Markdown).
---

# Roadmap Discovery

Autonomous codebase analysis to identify improvement opportunities without user interaction.

## Contextd Integration (Optional)

If contextd MCP is available:
- `repository_index` for semantic search
- `branch_create/return` for isolated lens analysis
- `memory_record` for discovery persistence
- `memory_search` to compare with previous runs (trend analysis)
- Track which findings were addressed across sessions

If contextd is NOT available:
- Use Glob/Grep for pattern analysis
- Run lenses sequentially
- Output to terminal or `.claude/discovery/`

## When to Use

- **Pre-session:** Run before brainstorming for context
- **Onboarding:** Part of `/init` setup flow
- **On-demand:** Via `/discover` command
- **Trend tracking:** Compare current state to previous discovery runs

## Lenses

Filter analysis by concern:

| Lens | Focus Area |
|------|------------|
| `security` | OWASP Top 10, auth gaps, injection risks, secrets exposure, dependency vulnerabilities |
| `quality` | Test coverage, complexity, duplication, error handling |
| `perf` | N+1 queries, missing indices, unbounded operations, memory leaks |
| `docs` | Missing README sections, outdated comments, API gaps |
| `dx` | Onboarding friction, documentation gaps, tooling issues, contributor barriers |
| `compliance` | License audit, regulatory requirements, audit trails, data handling |
| `a11y` | WCAG compliance, a11y testing gaps, semantic HTML, ARIA usage |
| `all` | Run all lenses (default) |

### Lens Categories

**Core Lenses** (always recommended):
- `security`, `quality`, `perf`, `docs`

**Extended Lenses** (context-dependent):
- `dx` - Run for open source or team projects
- `compliance` - Run for enterprise or regulated industries
- `a11y` - Run for web/mobile UI projects

## Severity Framework

Classify findings by impact:

| Severity | Description | Action |
|----------|-------------|--------|
| CRITICAL | Security vulnerabilities, data loss risks | Immediate attention |
| MAJOR | Performance bottlenecks, significant UX issues | High priority |
| MINOR | Code smell, minor gaps | Normal priority |
| SUGGESTION | Nice-to-haves, polish items | Backlog |

## Complexity Scoring

Each finding includes complexity estimation for prioritization:

### Per-Finding Complexity

| Complexity | Effort | Typical Scope |
|------------|--------|---------------|
| TRIVIAL | < 1 hour | Single line change, config update |
| LOW | 1-4 hours | Single file, isolated change |
| MEDIUM | 1-2 days | Multiple files, some testing needed |
| HIGH | 3-5 days | Architectural change, significant refactor |
| EPIC | 1+ weeks | Major feature, cross-cutting concern |

### Effort vs Impact Matrix

Findings are classified into quadrants for prioritization:

```
                    HIGH IMPACT
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
    │   STRATEGIC        │   QUICK WINS       │
    │   (Plan for        │   (Do First)       │
    │    later)          │                    │
    │                    │                    │
────┼────────────────────┼────────────────────┼────
    │                    │                    │   LOW
HIGH│                    │                    │   EFFORT
EFFORT                   │                    │
    │   AVOID            │   FILL-INS         │
    │   (Deprioritize)   │   (Do when idle)   │
    │                    │                    │
    └────────────────────┼────────────────────┘
                         │
                    LOW IMPACT
```

**Quick Wins:** High impact, low effort - prioritize these
**Strategic:** High impact, high effort - plan and schedule
**Fill-ins:** Low impact, low effort - do during downtime
**Avoid:** Low impact, high effort - deprioritize or eliminate

## Execution Flow

### 1. Index Repository

```
mcp__contextd__repository_index(
  path: ".",
  exclude_patterns: ["node_modules/**", "vendor/**", ".git/**"]
)
```

### 2. Run Lens Analyzers

**Use Task tool for parallel execution when running multiple lenses.**

#### Parallel Execution Pattern

```
# Dispatch lenses in parallel via Task tool
Task(agent: "lens-analyzer", prompt: "Run security lens on <path>")
Task(agent: "lens-analyzer", prompt: "Run quality lens on <path>")
Task(agent: "lens-analyzer", prompt: "Run perf lens on <path>")
Task(agent: "lens-analyzer", prompt: "Run docs lens on <path>")
# Wait for all to complete, then aggregate
```

#### Core Lenses

**Security:**
- `Grep: password|secret|api_key|token|credential` → Secrets exposure
- `Grep: innerHTML|dangerouslySetInnerHTML` → DOM injection risks
- `Grep: sql.*\+|"SELECT.*\$|'SELECT.*\$` → SQL injection
- `Grep: crypto\.createHash\(['"]md5|sha1` → Weak cryptography
- Check `package.json`/`requirements.txt` for known vulnerable deps
- Scan for hardcoded IPs, URLs, or credentials
- OWASP Top 10 pattern matching

**Quality:**
- `Grep: catch\s*\(\w*\)\s*\{\s*\}` → Empty catch blocks
- `Grep: TODO|FIXME|HACK|XXX` → Technical debt markers
- Test file ratio analysis → Coverage gaps
- Cyclomatic complexity estimation → Overly complex functions
- Duplicate code detection → DRY violations

**Perf:**
- `Grep: for.*SELECT|\.find\(\)(?!.*limit)` → N+1 queries
- `Grep: \.forEach.*await|for.*await` → Sequential async in loops
- `Grep: JSON\.parse\(.*JSON\.stringify` → Unnecessary serialization
- Missing database indices check
- Unbounded array/list operations

**Docs:**
- `Glob: README.md` + section check → Required sections present
- API endpoint documentation → OpenAPI/Swagger coverage
- Inline comment quality → Complex code documented
- CHANGELOG.md existence and currency

#### Extended Lenses

**Developer Experience (dx):**
- `Glob: CONTRIBUTING.md` → Contributor guide exists
- Setup script analysis → Onboarding friction assessment
- `Grep: # TODO: document|needs docs` → Documentation debt
- Dev tooling audit → Linter, formatter, pre-commit hooks
- Local development environment → docker-compose, devcontainer
- Error message quality → Helpful vs cryptic errors

**Compliance:**
- `Glob: LICENSE*` → License file exists and valid
- Dependency license audit → Compatible licenses
- `Grep: PII|GDPR|HIPAA|SOC2` → Compliance markers
- Audit logging presence → Sensitive operations logged
- Data retention policies → Documented and implemented
- Third-party data sharing → Privacy policy alignment

**Accessibility (a11y):**
- `Grep: <img(?![^>]*alt=)` → Images without alt text
- `Grep: onClick(?![^}]*onKeyDown)` → Click without keyboard handler
- `Grep: role=|aria-` → ARIA usage patterns
- Semantic HTML analysis → div/span overuse vs semantic elements
- Color contrast indicators → Hardcoded colors to check
- Focus management → tabindex usage, focus traps
- `Glob: *.test.*|*.spec.*` → a11y test coverage (jest-axe, cypress-axe)

### 3. Aggregate Findings

Combine results across lenses:

```json
{
  "project": {
    "name": "<project>",
    "path": "<path>",
    "analyzed_at": "<timestamp>",
    "discovery_version": "2.0"
  },
  "summary": {
    "total_findings": "<count>",
    "by_severity": {
      "CRITICAL": "<count>",
      "MAJOR": "<count>",
      "MINOR": "<count>",
      "SUGGESTION": "<count>"
    },
    "by_lens": {
      "security": "<count>",
      "quality": "<count>",
      "perf": "<count>",
      "docs": "<count>",
      "dx": "<count>",
      "compliance": "<count>",
      "a11y": "<count>"
    },
    "quick_wins": "<count>",
    "total_effort_days": "<estimated>"
  },
  "trend": {
    "previous_run": "<timestamp or null>",
    "delta": {
      "total": "<+/- count>",
      "CRITICAL": "<+/- count>",
      "MAJOR": "<+/- count>"
    },
    "resolved_since_last": ["<finding_ids>"],
    "new_since_last": ["<finding_ids>"],
    "regressions": ["<finding_ids that reappeared>"]
  },
  "findings": [
    {
      "id": "find_001",
      "lens": "security",
      "severity": "MAJOR",
      "title": "<short title>",
      "description": "<detailed description>",
      "location": "<file:line or pattern>",
      "recommendation": "<how to fix>",
      "effort": "low|medium|high",
      "complexity": "TRIVIAL|LOW|MEDIUM|HIGH|EPIC",
      "impact": "low|medium|high",
      "quadrant": "quick-win|strategic|fill-in|avoid",
      "confidence": "0.0-1.0",
      "references": ["<links to docs/standards>"],
      "first_seen": "<timestamp>",
      "occurrences": "<count if recurring>"
    }
  ]
}
```

### 4. Trend Analysis

Compare current run to previous discovery results:

```
# Load previous discovery from contextd
mcp__contextd__memory_search(
  project_id: "<project>",
  query: "discovery analysis",
  limit: 1
)

# Compare findings
- New findings: Present now, absent before
- Resolved findings: Absent now, present before
- Regressions: Resolved previously, reappeared now
- Persistent: Present in both runs
```

**Trend Metrics:**
- **Improvement rate:** % of findings resolved since last run
- **Regression rate:** % of resolved findings that reappeared
- **Velocity:** Average findings resolved per week
- **Accumulation:** New findings introduced per week

### 5. Store in contextd

```
mcp__contextd__memory_record(
  project_id: "<project>",
  title: "Discovery: <lens> analysis",
  content: "Found <total> issues. CRITICAL: <n>, MAJOR: <n>.
            Top findings: <list top 3>.
            Trend: <+/-n> since last run. <n> quick wins available.",
  outcome: "success",
  tags: ["roadmap-discovery", "<lens>", "<date>"]
)
```

### 6. Create Artifacts (Optional)

Based on output format flags, generate artifacts.

## Output Formats

| Flag | Output |
|------|--------|
| (default) | Summary to terminal |
| `--create-issues` | Create GitHub Issues |
| `--jira` | JIRA import CSV format |
| `--markdown` | Markdown roadmap document |
| `--json` | Full JSON output |
| `--contextd-only` | Store only, no terminal output |

### GitHub Issues Format

When `--create-issues` flag is set:

```bash
gh issue create \
  --title "[<severity>][<lens>] <title>" \
  --label "<lens>,discovery,<severity-lowercase>" \
  --body "## Description
<description>

## Location
\`<file:line>\`

## Recommendation
<recommendation>

## Metadata
- **Effort:** <effort>
- **Complexity:** <complexity>
- **Impact:** <impact>
- **Confidence:** <confidence>

## References
<references>

---
*Generated by roadmap-discovery v2.0*"
```

**Integration with github-planning skill:**
- Use `github-planning` skill for epic creation when findings > 10
- Group related findings into parent issues
- Apply project board automation labels

### JIRA Import Format

When `--jira` flag is set, generate CSV:

```csv
Summary,Description,Issue Type,Priority,Labels,Story Points
"[SECURITY] <title>","<description>\n\nRecommendation: <rec>",Task,<priority>,discovery;security,<points>
```

**Priority Mapping:**
- CRITICAL → Highest
- MAJOR → High
- MINOR → Medium
- SUGGESTION → Low

**Story Points Mapping:**
- TRIVIAL → 1
- LOW → 2
- MEDIUM → 5
- HIGH → 8
- EPIC → 13

### Markdown Roadmap Format

When `--markdown` flag is set:

```markdown
# Discovery Roadmap - <project>

Generated: <timestamp>
Previous Run: <timestamp or "First run">

## Executive Summary

- **Total Findings:** <n>
- **Critical Issues:** <n> (immediate attention required)
- **Quick Wins:** <n> (high impact, low effort)
- **Trend:** <+/-n> since last run

## Quick Wins (Do First)

| Finding | Lens | Effort | Location |
|---------|------|--------|----------|
| <title> | <lens> | <effort> | <location> |

## Critical Issues

### <title>
- **Severity:** CRITICAL
- **Location:** `<file:line>`
- **Description:** <description>
- **Recommendation:** <recommendation>

## By Category

### Security (<n> findings)
...

### Quality (<n> findings)
...

## Trend Analysis

| Metric | Value |
|--------|-------|
| Resolved since last | <n> |
| New since last | <n> |
| Regressions | <n> |
| Improvement rate | <n>% |

## Appendix: All Findings

<full findings table>
```

## Integration Points

### With /brainstorm
Run discovery before brainstorming to inform design:
```
1. /discover --lens all
2. Review findings
3. /brainstorm with context
```

### With /init
Run discovery during project setup:
```
/init            # Set up project, then run /discover
/discover --lens security,quality
```

### With github-planning
Create structured GitHub artifacts from findings:
```
/discover --lens all
# For many findings, use github-planning to create epic
/plan --from-discovery
```

### Pre-session Hook
Can be configured as SessionStart hook for automatic context.

### Cross-Project Comparison
With contextd, compare findings across projects:
```
mcp__contextd__memory_search(
  query: "discovery CRITICAL",
  limit: 10
)
# Aggregate patterns across all indexed projects
```

## Limitations

- Non-interactive: Cannot ask clarifying questions
- Pattern-based: May have false positives
- Point-in-time: Reflects current state only
- Trend analysis requires previous contextd records

## Attribution

Adapted from Auto-Claude roadmap_discovery and ideation agents.
See CREDITS.md for full attribution.

---

## Pre-Flight (MANDATORY)

**BEFORE running ANY analysis:**

```
1. Check repository index freshness:
   - If no index exists: mcp__contextd__repository_index(path: ".")
   - If HEAD changed since last index: re-index
   - Never skip indexing - stale data = stale findings

2. mcp__contextd__memory_search(
     project_id: "<project>",
     query: "discovery analysis <date>"
   )
   → Load recent discovery results for trend analysis

3. Determine lenses:
   - Default: all (run security, quality, perf, docs, dx, compliance, a11y)
   - If --lens specified: parse comma-separated list
   - If --core: run only security, quality, perf, docs
   - NEVER ask user which lens - use default or argument

4. Set finding caps:
   - Max 10 findings per lens per severity
   - Prioritize by confidence and impact

5. Check output format:
   - Default: terminal summary
   - Parse --create-issues, --jira, --markdown, --json flags
```

**Do NOT ask the user any questions during discovery. This is autonomous analysis.**

---

## Mandatory Checklist

**EVERY discovery invocation MUST complete ALL steps:**

### Analysis Phase:
- [ ] Complete pre-flight (index check, memory search)
- [ ] Run ALL specified lenses (default: all 7)
- [ ] Use Task tool for parallel lens execution when possible
- [ ] For each lens, execute analysis (see Run Lens Analyzers section)
- [ ] Apply severity classification to each finding
- [ ] Apply complexity scoring to each finding
- [ ] Calculate effort vs impact quadrant
- [ ] Apply confidence scoring

### Quality Control:
- [ ] Cap findings (max 10 per lens per severity)
- [ ] Group similar findings
- [ ] Remove duplicates
- [ ] Sort by severity then confidence
- [ ] Identify quick wins (high impact + low effort)

### Trend Analysis:
- [ ] Load previous discovery results from contextd
- [ ] Calculate delta (new, resolved, regressions)
- [ ] Compute improvement metrics
- [ ] Flag regressions prominently

### Output Phase:
- [ ] Generate summary with counts by severity and lens
- [ ] List quick wins first
- [ ] List top findings (max 10 total)
- [ ] If --create-issues: create GitHub Issues (respecting --min-severity)
- [ ] If --jira: generate JIRA CSV
- [ ] If --markdown: generate roadmap document
- [ ] **RECORD to contextd** (see Storage section)

### Error Handling:
- [ ] If a lens fails, continue with other lenses
- [ ] Report partial success with failure details
- [ ] Never fail entire discovery for one lens error

**Discovery is NOT complete until contextd recording is done.**

---

## Autonomy Requirements (Critical)

**Discovery is 100% non-interactive. NEVER:**

- Ask "Which lens should I analyze?"
- Ask "Should I create issues for this?"
- Ask "Is this finding valid?"
- Wait for user confirmation
- Request clarification on severity

**ALWAYS:**

- Make reasonable inferences based on patterns
- Use confidence scores instead of asking
- Run with defaults if no arguments provided
- Complete fully without user input
- Record ALL findings (contextd stores everything)
- Use Task tool for parallel execution when appropriate

**If you catch yourself about to ask a question: STOP. Make a decision and document it.**

---

## Red Flags - STOP and Reconsider

If you're thinking any of these, you're about to violate the skill:

| Thought | Reality |
|---------|---------|
| "Which lens should I run?" | Default is ALL. Run all 7 lenses. |
| "Should I ask about this finding?" | No. Use confidence scoring, not questions. |
| "Repository isn't indexed" | Index it yourself. mcp__contextd__repository_index. |
| "There are 100+ findings" | Cap at 10 per lens per severity. Quality > quantity. |
| "This lens failed, abort" | Continue other lenses. Partial success is valid. |
| "I'll skip contextd, just terminal output" | contextd recording is MANDATORY. |
| "User didn't specify lens" | Default = all. Don't ask. |
| "Not sure about severity" | Make your best judgment. Document rationale. |
| "Should I run lenses sequentially?" | Use Task tool for parallel execution. |
| "No previous run to compare" | Skip trend analysis, note "First run". |

---

## Common Mistakes

| Mistake | Correct Approach |
|---------|------------------|
| Asking user which lens to run | Default is ALL. Use --lens argument if specified. |
| Failing when repo not indexed | Auto-index with repository_index before analysis. |
| Returning 100+ findings | Cap at 10 per lens per severity. Prioritize by impact. |
| Aborting on lens failure | Continue with working lenses. Report partial success. |
| Skipping contextd | memory_record is mandatory for every discovery run. |
| Waiting for user input | Discovery is autonomous. Make decisions, document them. |
| Using stale index | Check if HEAD changed. Re-index if needed. |
| Treating all findings equally | Apply severity, complexity, AND confidence scoring. |
| Running lenses sequentially | Use Task tool for parallel lens execution. |
| Ignoring quick wins | Always identify and highlight quick wins first. |
| Skipping trend analysis | Always compare to previous run if available. |

---

## Self-Sufficiency Protocol

**Discovery must handle its own prerequisites:**

### Missing Index
```
IF mcp__contextd__semantic_search returns no results:
  mcp__contextd__repository_index(
    path: ".",
    exclude_patterns: ["node_modules/**", "vendor/**", ".git/**", "dist/**"]
  )
  THEN proceed with analysis
```

### Stale Index
```
IF git log -1 --format=%H != last_indexed_commit:
  mcp__contextd__repository_index(path: ".")
  THEN proceed with analysis
```

### Partial Lens Failure
```
IF security_analyzer throws error:
  record error in findings: "Security analysis failed: <reason>"
  continue with quality, perf, docs, dx, compliance, a11y analyzers
  include partial results in output
  note failure in summary
```

### No Previous Discovery
```
IF mcp__contextd__memory_search returns no previous discovery:
  skip trend analysis
  note "First discovery run" in output
  proceed with full analysis
```

---

## Confidence Scoring

**Apply confidence to each finding:**

| Confidence | Meaning | Action |
|------------|---------|--------|
| HIGH (0.8-1.0) | Pattern clearly matches, minimal false positive risk | Include in top findings |
| MEDIUM (0.5-0.79) | Likely issue, but context matters | Include with caveat |
| LOW (0.2-0.49) | Possible issue, may be false positive | Include only if CRITICAL severity |
| VERY LOW (<0.2) | Likely false positive | Exclude from output |

**When in doubt about severity or confidence: bias toward including the finding with documented uncertainty rather than asking the user.**

---

## Finding Caps (Quality Control)

**Maximum findings per category:**

| Level | Cap | Rationale |
|-------|-----|-----------|
| Per lens, per severity | 10 | Prevents flood of minor issues |
| Total per lens | 25 | Focus on most impactful |
| Total output | 50 | Actionable, not overwhelming |
| Top findings summary | 10 | User can digest quickly |
| Quick wins highlight | 5 | Immediate action items |

**Prioritization order:**
1. Severity (CRITICAL > MAJOR > MINOR > SUGGESTION)
2. Quadrant (quick-win > strategic > fill-in > avoid)
3. Confidence (HIGH > MEDIUM > LOW)
4. Effort (LOW > MEDIUM > HIGH for same severity)

**If over cap: drop lowest priority items, note "N additional findings omitted"**

---

## Extended Lens Details

### Security Lens - OWASP Top 10 Coverage

| OWASP Category | Detection Pattern |
|----------------|-------------------|
| A01: Broken Access Control | Auth middleware gaps, missing role checks |
| A02: Cryptographic Failures | Weak hashing, hardcoded secrets |
| A03: Injection | SQL concatenation, innerHTML, exec patterns |
| A04: Insecure Design | Missing rate limits, no input validation |
| A05: Security Misconfiguration | Debug mode, default creds, verbose errors |
| A06: Vulnerable Components | Outdated deps, known CVEs |
| A07: Auth Failures | Weak passwords, missing MFA |
| A08: Data Integrity | Missing signatures, untrusted deserialization |
| A09: Logging Failures | Missing audit logs, sensitive data in logs |
| A10: SSRF | Unvalidated URLs, fetch with user input |

### Compliance Lens - License Categories

| License Type | Compatibility | Action |
|--------------|---------------|--------|
| MIT, BSD, Apache 2.0 | Permissive | Generally safe |
| GPL, LGPL, AGPL | Copyleft | Review obligations |
| Proprietary | Restricted | Verify license agreement |
| Unknown | Risk | Investigate before use |

### Accessibility Lens - WCAG Levels

| Level | Requirement | Examples |
|-------|-------------|----------|
| A (Minimum) | Basic accessibility | Alt text, keyboard nav |
| AA (Recommended) | Enhanced | Color contrast, focus visible |
| AAA (Optimal) | Full accessibility | Sign language, extended audio |

---

## Contextd Integration Details

### Recording Findings

```
# Record summary for quick retrieval
mcp__contextd__memory_record(
  project_id: "<project>",
  title: "Discovery Run <date>",
  content: "<summary JSON>",
  outcome: "success",
  tags: ["discovery", "<date>"]
)

# Record individual critical findings
FOR each CRITICAL finding:
  mcp__contextd__memory_record(
    project_id: "<project>",
    title: "CRITICAL: <finding title>",
    content: "<finding details>",
    outcome: "pending",
    tags: ["discovery", "critical", "<lens>"]
  )
```

### Tracking Resolution

When a finding is addressed:
```
mcp__contextd__memory_outcome(
  memory_id: "<finding_memory_id>",
  outcome: "resolved",
  notes: "Fixed in commit <sha>"
)
```

### Cross-Project Analysis

```
# Find common patterns across projects
mcp__contextd__memory_search(
  query: "CRITICAL security",
  limit: 50
)
# Aggregate findings by type to identify systemic issues
```
