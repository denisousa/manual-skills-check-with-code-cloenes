<!-- tribos-dev/wakanda-ia-sdd:sdd-kit/.claude/skills/sdd-code-reviewer/SKILL.md | https://github.com/tribos-dev/wakanda-ia-sdd/blob/HEAD/sdd-kit/.claude/skills/sdd-code-reviewer/SKILL.md -->

---
name: sdd-code-reviewer
stack: core
description: Code review specialist for SDD Kit. This is a SKILL (invoke via Skill tool, NOT Task/subagent). Use after implementing code during /sdd.build to run quality checks. BLOCKING gate - feature cannot proceed until code review passes. **TRIGGER ON** code review, quality gate, implementation review, PR review.
---

# SDD Code Reviewer

> **SKILL**: Quality gate for code review. Invoke with `Skill("sdd-code-reviewer")`. Use after each task in /sdd.build and before /sdd.finish.

---

## When to Use

1. **After Task Implementation** (`/sdd.build`) - Run after each task
2. **Before Feature Completion** (`/sdd.finish`) - BLOCKING gate
3. **On Demand** - When user requests code review

---

## Review Process

### Step 1: Identify Modified Files

```bash
# Get list of modified files
git diff --name-only HEAD~1
# Or for all changes in feature
git diff --name-only main...HEAD
```

### Step 2: Check with code review tool (if available)

If code review tool is available:
```
mcp__code review tool__code_review_instructions(application_name)
mcp__code review tool__pr_review_search(application_name, pr_number)
```

### Step 3: Manual Review Checklist

If MCP not available, use this checklist:

#### Code Quality
- [ ] Clear, readable code
- [ ] Proper naming conventions (camelCase, PascalCase as per lang)
- [ ] No code duplication (DRY principle)
- [ ] Functions are focused (single responsibility)
- [ ] No commented-out code
- [ ] No TODO/FIXME without tracking

#### Security (Critical)
- [ ] No hardcoded secrets (API keys, passwords, tokens)
- [ ] Input validation present for all user inputs
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (output encoding)
- [ ] No sensitive data in logs

#### Error Handling
- [ ] Exceptions properly caught and handled
- [ ] Meaningful error messages
- [ ] No swallowed exceptions (empty catch blocks)
- [ ] Graceful degradation where appropriate

#### Testing
- [ ] Unit tests for new code
- [ ] Test coverage adequate (>80%)
- [ ] Edge cases covered
- [ ] Integration tests where needed

####  Compliance
- [ ] Dockerfile uses `your-registry/base-image
- [ ] Dockerfile.runtime uses `your-registry/base-image
- [ ] /ping endpoint implemented
- [ ] Health checks configured

#### Documentation
- [ ] Public APIs documented
- [ ] Complex logic commented
- [ ] README updated if needed

---

## Review Output Format

```markdown
## Code Review Results

### Summary
- Files Reviewed: N
- Issues Found: N
- Severity Breakdown: X critical, Y major, Z minor

### Critical Issues (MUST Fix)
1. **[File:Line]** - [Issue description]
   - **Problem**: [What's wrong]
   - **Fix**: [How to fix it]
   - **Example**:
     ```
     // Before (wrong)
     ...
     // After (correct)
     ...
     ```

### Major Issues (Should Fix)
1. **[File:Line]** - [Issue description]
   - **Suggestion**: [Improvement]

### Minor Issues (Nice to Have)
1. **[File:Line]** - [Improvement idea]

### Verdict
[ ] APPROVED - Ready to proceed
[ ] CHANGES REQUESTED - Fix critical/major issues first
```

---

## Common Issues to Flag

### Security
| Pattern | Issue | Severity |
|---------|-------|----------|
| `password = "..."` | Hardcoded password | CRITICAL |
| `"SELECT * FROM " + input` | SQL injection | CRITICAL |
| `innerHTML = userInput` | XSS vulnerability | CRITICAL |
| `console.log(password)` | Sensitive data in logs | MAJOR |

### Performance
| Pattern | Issue | Severity |
|---------|-------|----------|
| Query in loop | N+1 query problem | MAJOR |
| `String +=` in loop | String concatenation | MAJOR |
| Missing index hint | Potential slow query | MINOR |

### Code Quality
| Pattern | Issue | Severity |
|---------|-------|----------|
| Function > 50 lines | Too long, split it | MINOR |
| > 3 parameters | Consider object param | MINOR |
| Empty catch block | Swallowed exception | MAJOR |

---

## Important Rules

1. **Be Specific**: Point to exact files and lines
2. **Propose Fixes**: Don't just identify, suggest solutions
3. **Prioritize**: Critical > Major > Minor
4. **Be Constructive**: Focus on improvement, not criticism
5. **Blocking Gate**: NEVER approve with critical issues
6. **ALL findings must be fixed**: Minor issues are NOT optional

---

## Verdict Output (MANDATORY)

> **v2.0.0**: After completing the review, you MUST write a verdict file.

### Verdict File Location

```
sdd/wip/<feature>/verdicts/code_review.json
```

### Write Verdict After Review

**After completing all checks, create the verdict file**:

```json
{
  "skill": "sdd-code-reviewer",
  "verdict": "APPROVED",
  "findings": {
    "critical": 0,
    "major": 0,
    "minor": 0
  },
  "files_reviewed": 15,
  "timestamp": "2026-01-19T12:00:00Z"
}
```

### Verdict Values

| Verdict | Condition | Task Completion |
|---------|-----------|-----------------|
| `APPROVED` | 0 critical, 0 major, 0 minor | Allowed |
| `CAN_PROCEED_WITH_WARNINGS` | 0 critical, 0 major, minor ≤ 3 | Allowed |
| `CANNOT_PROCEED` | Any critical OR any major | BLOCKED |

### Verdict Writing Instructions

1. **Create verdicts directory** if it doesn't exist:
   ```bash
   mkdir -p sdd/wip/<feature>/verdicts
   ```

2. **Write the verdict file** with current findings count

3. **Verdict determines if Layer 3 task can be completed**:
   - `APPROVED` → Task can be marked complete
   - `CANNOT_PROCEED` → Must fix issues and re-run this skill

> **CRITICAL**: The enforcement hook checks this file before allowing task completion.
