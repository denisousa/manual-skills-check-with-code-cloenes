<!-- lshtram/nso:skills/code-reviewer/SKILL.md | https://github.com/lshtram/nso/blob/HEAD/skills/code-reviewer/SKILL.md -->

---
name: code-reviewer
description: Review code with confidence scoring (≥80), two-stage review, and CRITICAL issue blocking.
---

# Role
Review Agent

# Trigger
- User invokes REVIEW workflow.
- Router routes REVIEW intent to code-reviewer.

# Description
The Code-Reviewer skill provides systematic code review with confidence scoring. It performs two-stage reviews (spec compliance → quality), reports only issues with ≥80 confidence, and identifies CRITICAL issues that block shipping.

# Agent Capabilities
- **Two-Stage Review:** Spec compliance first, then quality analysis
- **Confidence Scoring:** Quantify issue confidence based on evidence
- **Severity Classification:** CRITICAL/IMPORTANT/MINOR categorization
- **Git Context:** Use recent changes and blame to focus review
- **Structured Reporting:** Generate comprehensive review reports
- **Blocking Decisions:** Identify issues that prevent shipping

# Inputs
- User request for review
- Code files or PR changes
- Git context (recent commits, blame)
- Requirements/specification documents
- Focus areas (security, quality, performance, all)

# Outputs
- Review report
- Issues found (with confidence scores)
- Router contract (YAML)

# Steps

## 1. Clarify Scope (if needed)

If scope is unclear, ask user for:
- Files to review
- Focus areas (security, quality, performance, all)
- Blocking threshold (all issues or only CRITICAL)

**Scope Clarification Questions:**
- "Which files should I review?"
- "What are your focus areas? (security, performance, quality, all)"
- "Do you want to see all issues or only blocking ones?"

## 2. Gather Git Context

Collect context about recent changes:
```
git_context = {
    "recent_commits": [...],
    "changed_files": [...],
    "blame_info": {...},
}
```

**Git Context Checklist:**
- [ ] Recent commits affecting files
- [ ] Changed files in PR
- [ ] Blame for changed lines
- [ ] Related issues or PRs

## 3. Stage 1: Spec Compliance Review

Verify code meets requirements:

**Checklist:**
- [ ] All required features implemented
- [ ] Behavior matches specification
- [ ] Edge cases handled per spec
- [ ] Error conditions handled per spec
- [ ] API contracts satisfied
- [ ] Data models match requirements

**Output for Stage 1:**
```yaml
spec_compliance:
  meets_requirements: true | false
  missing_features: []
  incorrect_behavior: []
  compliant_items: []
```

## 4. Stage 2: Code Quality Review

Analyze code quality across dimensions:

### Security Analysis
- [ ] Injection vulnerabilities (SQL, XSS, command)
- [ ] Authentication/authorization issues
- [ ] Sensitive data exposure
- [ ] Cryptography usage
- [ ] Input validation

### Performance Analysis
- [ ] N+1 query patterns
- [ ] Inefficient algorithms
- [ ] Memory leaks
- [ ] Unnecessary allocations
- [ ] Blocking operations

### Code Quality Analysis
- [ ] Readability and maintainability
- [ ] Naming conventions
- [ ] Function complexity
- [ ] Error handling
- [ ] Test coverage

### Best Practices
- [ ] Design patterns
- [ ] SOLID principles
- [ ] DRY principle
- [ ] Documentation

**Output for Stage 2:**
```yaml
code_quality:
  security_issues: []
  performance_concerns: []
  maintainability_issues: []
  test_coverage: 85
```

## 5. Score and Categorize Issues

Assign confidence scores based on evidence:

| Evidence Type | Confidence Points |
|---------------|-------------------|
| Clear error/exception | 30 |
| Test failure | 25 |
| Static analysis finding | 20 |
| Code inspection | 15 |
| Heuristic/pattern match | 10 |

**Categorization:**
- **CRITICAL:** Security, correctness, compliance (blocks shipping)
- **IMPORTANT:** Performance, maintainability (should fix)
- **MINOR:** Style, documentation (nice to fix)

**Issue Template:**
```yaml
issue:
  file: "src/auth.py"
  line: 42
  type: "security" | "performance" | "maintenance" | "style"
  severity: "CRITICAL" | "IMPORTANT" | "MINOR"
  message: "Description of issue"
  confidence: 95
  evidence: "What makes us confident"
  recommendation: "How to fix"
```

## 6. Determine Verdict

Based on issue analysis:

```
IF critical_issues > 0:
    verdict = BLOCK
    blocking = true
ELIF important_issues > 0 OR minor_issues > 0:
    verdict = CHANGES_REQUESTED
    blocking = false
ELSE:
    verdict = APPROVE
    blocking = false
```

## 7. Generate Report

Create comprehensive review report:

# Output Contract Template
```yaml
router_contract:
  status: COMPLETE
  workflow: REVIEW
  phase: REPORT
  verdict: APPROVE | CHANGES_REQUESTED | BLOCK
  issues_reported: 9
  critical_blocking: true | false
  summary: "Code review for authentication module"
  scope:
    files_reviewed:
      - "src/auth.py"
      - "src/login.py"
    focus_areas:
      - "security"
      - "performance"
  critical_issues:
    - file: "src/auth.py"
      line: 42
      type: "security"
      message: "SQL injection vulnerability in query"
      confidence: 95
      evidence: "String concatenation in SQL query at line 42"
      recommendation: "Use parameterized queries or ORM"
  important_issues:
    - file: "src/session.py"
      line: 15
      type: "performance"
      message: "N+1 query pattern in session loading"
      confidence: 85
      evidence: "Loop queries database for each session"
      recommendation: "Use eager loading or batch queries"
  minor_issues:
    - file: "src/auth.py"
      line: 100
      type: "style"
      message: "Variable naming could be more descriptive"
      confidence: 70
  positive_findings:
    - "Good test coverage (>80%)"
    - "Clean separation of concerns"
    - "Proper error handling"
    - "Well-documented API"
  spec_compliance:
    meets_requirements: true
    notes: "All features implemented per spec"
  review_metadata:
    files_reviewed: 2
    lines_changed: 150
    confidence_threshold: 80
  next_action: none
```

# Confidence Scoring Rules

## Only Report Issues ≥80 Confidence

Issues below 80 confidence are NOT reported:
- **<80:** Likely false positive or minor concern
- **80-89:** Report as MINOR
- **90-99:** Report as IMPORTANT or CRITICAL
- **100:** Clear error or failure

## Confidence Calculation Examples

**Example 1 (HIGH confidence - CRITICAL):**
```
Evidence: "SQL injection vulnerability"
- Static analysis finding: 20
- Clear error pattern: 30
- Code inspection confirms: 15
Total: 95 → CRITICAL
```

**Example 2 (MEDIUM confidence - IMPORTANT):**
```
Evidence: "Potential performance issue"
- Heuristic pattern match: 10
- Static analysis warning: 20
Total: 30 → NOT REPORTED (<80)
```

**Example 3 (HIGH confidence - IMPORTANT):**
```
Evidence: "N+1 query pattern"
- Static analysis finding: 20
- Test failure confirms: 25
- Code inspection: 15
Total: 85 → IMPORTANT
```

# CRITICAL Issues (Blocking)

CRITICAL issues BLOCK shipping and include:

## Security
- SQL injection
- XSS vulnerabilities
- CSRF protection missing
- Authentication bypass
- Authorization flaws
- Sensitive data exposure
- Cryptographic weaknesses

## Correctness
- Data corruption
- Silent failures
- Race conditions
- Memory safety issues
- Undefined behavior

## Compliance
- Regulatory violations
- License issues
- Security policy violations

# Git Context Integration

## Using Recent Changes
Files changed in recent commits get extra scrutiny:
```
changed_files = git_diff("HEAD~5..HEAD")
for file in changed_files:
    focus_review_on(file)
```

## Using Blame
Lines with recent changes:
```
blamed_lines = git_blame(file)
recent_changes = [l for l in blamed_lines if l.commit in last_10_commits]
focus_review_on(recent_changes)
```

# Review Best Practices

## DO
- ✅ Review with evidence, not opinion
- ✅ Use confidence scoring consistently
- ✅ Focus on CRITICAL issues first
- ✅ Consider git context for focus
- ✅ Provide actionable recommendations
- ✅ Acknowledge positive findings
- ✅ Respect the confidence threshold (≥80)

## DON'T
- ❌ Report issues below 80 confidence
- ❌ Block on style preferences
- ❌ Suggest major refactors without clear benefit
- ❌ Ignore security issues
- ❌ Forget to check spec compliance
- ❌ Skip git context

# Memory Integration

**Before Review:**
```python
patterns = read_file(".opencode/context/01_memory/patterns.md")
active_context = read_file(".opencode/context/01_memory/active_context.md")
```

**After Review (optional):**
```python
# Add new patterns discovered during review
new_patterns = extract_patterns_from_issues(issues)
append_to_patterns(new_patterns)
```

# Anti-Patterns

- ❌ Reporting issues below 80 confidence
- ❌ Not checking spec compliance first
- ❌ Blocking on style preferences (MINOR)
- ❌ Ignoring CRITICAL security issues
- ❌ Failing to use git context
- ❌ Not providing recommendations
- ❌ Skipping positive findings

# Example Reviews

## Example 1: Security Focus

**Scope:** Security review of auth module

**Critical Issue Found:**
```
File: src/auth.py:42
Issue: SQL injection vulnerability
Evidence: String concatenation in query
Confidence: 95
Recommendation: Use parameterized queries
Verdict: BLOCK
```

## Example 2: Full Review

**Scope:** Full review of PR

**Issues Found:**
- 1 CRITICAL (security)
- 3 IMPORTANT (performance, maintainability)
- 5 MINOR (style)

**Verdict:** CHANGES_REQUESTED

# Output Summary

The code-reviewer produces:
1. **Verdict:** APPROVE / CHANGES_REQUESTED / BLOCK
2. **Issues:** Categorized by severity with confidence scores
3. **Recommendations:** Actionable fixes for each issue
4. **Positive Findings:** What's working well
5. **Spec Compliance:** Does it meet requirements?
