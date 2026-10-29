<!-- dinoudon/sdlc-skills:full/sdlc-adversarial-review/SKILL.md | https://github.com/dinoudon/sdlc-skills/blob/HEAD/full/sdlc-adversarial-review/SKILL.md -->

---
name: sdlc-adversarial-review
description: "Multi-agent PR review: 3 specialized reviewers (architecture, security, quality) run in parallel, orchestrator synthesizes findings and applies fixes. Includes DORA metrics, SLSA supply chain, AI-assisted review guardrails, threat modeling (STRIDE/ATT&CK), secure code patterns, compliance (SOC2/GDPR/HIPAA), OWASP API Top 10, container/IaC security, policy as code, zero trust, and supply chain security."
version: 4.8.0-moderate
author: Dinoudon
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [sdlc, code-review, pr-review, adversarial, multi-agent, security, architecture, dora, slsa, supply-chain, threat-modeling, compliance, owasp-api, zero-trust, policy-as-code]
    related_skills: [sdlc-architecture-design, sdlc-testing-qa, github-code-review, github-pr-workflow]
---

# Adversarial PR Review

3-agent parallel review system: architecture + security + quality reviewers run in parallel, orchestrator synthesizes and fixes.

## When to Use

- Reviews a PR before merging to main
- Wants thorough code review (architecture + security + quality)
- Has significant refactors to validate
- Needs security-sensitive changes reviewed

## When NOT to Use

- Trivial changes (typos, comments, version bumps)
- Draft PRs still in progress
- Hotfixes needing immediate merge

## Step 0: Automated Pre-Review

Run automated checks first — machines handle mechanical checks, humans handle design/nuance.

```bash
# Semgrep — fast pattern-based SAST
semgrep --config=auto --severity=ERROR --severity=WARNING .

# CodeQL — deep semantic analysis (if GitHub repo)
gh codeql analyze --language=javascript

# Trivy — dependency + secrets + IaC scanning
trivy fs --scanners vuln,secret,misconfig .
```

**Tool selection matrix:**

| Tool | Speed | Depth | Best for |
|------|-------|-------|----------|
| Semgrep | Fast | Medium | Security patterns, code standards, banned APIs |
| CodeQL | Slow | Deep | Cross-function taint analysis, SQLi/XSS/SSRF |
| Trivy | Fast | Medium | Dependency CVEs, exposed secrets, IaC misconfig |
| cosign/Sigstore | Fast | Low | Artifact signature verification, provenance |

### SLSA Supply Chain Review (Step 0b)

| SLSA Level | Requirement | Review Action |
|------------|-------------|---------------|
| L0 | No guarantees | Flag as risk — no provenance |
| L1 | Provenance exists | Verify build provenance attestation present |
| L2 | Hosted build platform | Verify builds run on hosted CI (not dev machines) |
| L3 | Hardened builds | Verify isolated, ephemeral build environments |

```bash
# Verify SLSA provenance (GitHub Actions)
gh attestation verify <artifact> --owner <org>

# SBOM verification
syft dir:. -o spdx-json > sbom.spdx.json
grype sbom:sbom.spdx.json

# Sigstore/cosign — verify container/artifact signatures
cosign verify --certificate-identity=<workflow> \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com <image>
```

**Supply chain checklist:**
- [ ] SBOM present and up-to-date (SPDX or CycloneDX)
- [ ] All dependencies pinned by hash, not mutable tags
- [ ] Build provenance attestation exists (SLSA L1+)
- [ ] No unpinned GitHub Actions (use `@sha256` not `@v1`)
- [ ] No `curl | bash` install patterns
- [ ] Lockfile committed and verified in CI
- [ ] Container images signed (Sigstore/cosign)
- [ ] No dependency confusion vectors

## Step 1: Spawn 3 Reviewers in Parallel

```python
tasks = [
    {
        "goal": "Review PR for architecture concerns",
        "context": """Focus on (priority order):
1. Design — right abstraction? Right pattern? (Clean/Hexagonal/DDD)
2. Dependencies — point inward? No circular deps?
3. Module boundaries — respected? Interfaces deep (high leverage)?
4. ADRs followed? If new pattern, flag for ADR creation.
5. Architecture fitness functions — would this change break any?

Report findings with severity and suggested fix.""",
        "toolsets": ["terminal", "file"]
    },
    {
        "goal": "Review PR for security vulnerabilities",
        "context": """Focus on (OWASP Top 10 + supply chain):
1. Injection vectors — SQL, NoSQL, LDAP, OS command
2. Auth issues — broken auth, session management, missing checks
3. Secrets exposure — hardcoded keys, tokens, passwords
4. Input validation — all user input sanitized? SSRF prevention?
5. XSS prevention — output encoding, CSP headers
6. CSRF protection — tokens on state-changing endpoints
7. Rate limiting — on auth and expensive endpoints
8. Dependency vulnerabilities — CVEs in dependencies
9. Supply chain — typosquatting, compromised packages

Report findings with severity and suggested fix.""",
        "toolsets": ["terminal", "file"]
    },
    {
        "goal": "Review PR for code quality",
        "context": """Focus on (Google review priority order):
1. Functionality — does this do what user needs? Edge cases?
2. Complexity — is code more complex than needed?
3. Tests — present, correct, maintainable? Coverage meets threshold?
4. Naming — clear, descriptive, consistent?
5. Comments — explain WHY, not WHAT
6. Error handling — comprehensive? Graceful degradation?
7. Performance — N+1 queries, unnecessary allocations, blocking I/O?
8. Documentation — API docs, inline docs, README updated for public changes?
9. PR description — clear what/why/how/testing sections?

Report findings with severity and suggested fix.""",
        "toolsets": ["terminal", "file"]
    }
]
# Run all 3 in parallel via delegate_task(tasks=tasks)
```

## Step 2: Orchestrator Synthesizes

1. Collect all 3 review reports
2. Deduplicate findings
3. Prioritize: critical > high > medium > low
4. Cross-reference: architecture finding may explain security finding
5. Apply DORA metrics: flag if PR >400 LOC (break up), >1 day open (unblock)

## Step 3: Apply Fixes

- If auto-fixable (lint, format, simple refactor) → fix and commit
- If needs human judgment → flag with comment
- If architectural → create issue for follow-up

## Step 4: Re-run CI

After fixes, push and verify CI passes. All automated checks must pass before merge.

## Review Templates

### Architecture Review
```
- [ ] Follows established patterns (Clean/Hexagonal/DDD)
- [ ] Dependencies point inward
- [ ] Module boundaries respected
- [ ] Interfaces are deep (high leverage)
- [ ] No circular dependencies
- [ ] ADRs followed (or new ADR created)
- [ ] Architecture fitness functions pass
- [ ] C4 model consistency maintained
```

### Security Review
```
- [ ] No hardcoded secrets (Trivy secrets scan)
- [ ] Input validation on all user input
- [ ] Auth checks on all endpoints
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (output encoding)
- [ ] CSRF protection (tokens)
- [ ] Rate limiting on auth/expensive endpoints
- [ ] Dependency vulnerabilities checked (Trivy/Dependabot)
- [ ] No SSRF vectors
- [ ] No path traversal vectors
```

### Quality Review
```
- [ ] Test coverage meets threshold (>80%)
- [ ] Edge cases tested
- [ ] Error handling comprehensive
- [ ] Naming descriptive and consistent
- [ ] No code duplication
- [ ] Performance considered (no N+1, no blocking I/O)
- [ ] Comments explain WHY, not WHAT
- [ ] PR size <400 LOC (or justified)
- [ ] Documentation updated
- [ ] CHANGELOG entry added (if user-facing change)
```

## DORA Velocity Metrics

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| PR to first review | <4 hours | <1 day | <1 week | >1 week |
| PR review to merge | <1 day | <3 days | <1 week | >1 week |
| PR size | <400 LOC | <800 LOC | <1500 LOC | >1500 LOC |
| PR lifetime | <1 day | <3 days | <1 week | >1 week |
| Reliability (DORA 2024) | >99.99% | >99.9% | >99% | <99% |

**Key findings:** Elite performers review in hours. Small batch size enables fastest review. Documentation quality correlates with high performance (DORA 2024). AI for review/testing outperforms AI for code generation alone.

## Code Review Culture

**Google reviewer priorities (in order):**
1. Design — right approach? Right abstraction?
2. Functionality — does this do what user needs?
3. Complexity — more complex than needed?
4. Tests — present, correct, maintainable?
5. Naming — clear, descriptive
6. Comments — explain WHY, not WHAT
7. Style — automate this away
8. Nit-picks — optional, prefix with "Nit:"

**Speed:** Respond within 4 hours. Small changes (<200 LOC) review in <1 hour. Never let PR sit >1 business day.

**Anti-Patterns:**
- **Bikeshedding** — arguing about trivial things
- **Rubber stamping** — approving without reading
- **Slow reviews** — PRs sitting for days
- **NIT overload** — too many optional comments
- **Design-by-committee** — conflicting reviewer opinions

## PR Template
```markdown
## What
[1-2 sentence description]

## Why
[Link to issue, business context]

## How
[Technical approach, key decisions]

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing done

## Checklist
- [ ] PR <400 LOC (or justified)
- [ ] No hardcoded secrets
- [ ] Error handling comprehensive
- [ ] Comments explain WHY
```

## Multi-Agent Review Architecture

```
PR opened
    │
    ▼
┌─────────────────────────────────────┐
│  Orchestrator Agent                  │
│  1. Run automated checks (Semgrep,   │
│     CodeQL, Trivy)                   │
│  2. Spawn 3 reviewers in parallel    │
│  3. Collect findings                  │
│  4. Deduplicate + prioritize         │
│  5. Apply auto-fixes                 │
│  6. Present unified review           │
└─────────┬───────────┬───────────┬────┘
          │           │           │
    ┌─────▼──┐  ┌─────▼──┐  ┌─────▼──┐
    │ Arch   │  │ Sec    │  │ Qual   │
    │ Review │  │ Review │  │ Review │
    └────────┘  └────────┘  └────────┘
```

## Stacked Diffs

Instead of one large PR, create chain of small dependent diffs (~100-300 lines each). Each is one logical change, can be reviewed independently. Enables parallel review.

**Tools:** ghstack (https://github.com/ezyang/ghstack), Graphite (https://graphite.dev/), Sapling (https://sapling-scm.com/)

## AI-Assisted Review

| Tool | Focus | Source |
|------|-------|--------|
| CodeRabbit | Line-by-line review, bug detection | https://coderabbit.ai/ |
| Sourcery | Python-focused, inline suggestions | https://sourcery.ai/ |
| GitHub Copilot PR review | Native GitHub integration | https://github.com/features/copilot |

**AI guardrails (DORA 2024):**
- Use AI for: first-pass review, test generation, security pattern detection, docs
- Don't use AI for: final architectural decisions, security-critical approval, blind acceptance
- Track AI suggestion acceptance rate — low = tool mismatch, high = rubber stamping
- All AI-generated code/comments need human approval before merge
- Require human reviewer for security-sensitive changes regardless of AI quality

## Pitfalls

1. **Don't run on every PR** — use for significant changes (>100 LOC, security-sensitive)
2. **Don't auto-fix architecture issues** — create issues
3. **Don't ignore medium/low findings** — accumulate into tech debt
4. **Don't skip re-running CI after fixes**
5. **Don't bikeshed** — focus on logic, not style
6. **Don't rubber stamp** — actually read the diff
7. **Don't let PRs sit** — respond within 4 hours
8. **Don't review for >60 minutes** — fatigue degrades quality
9. **Don't skip automated checks** — machines handle mechanical, humans handle design
10. **Don't mix abstraction levels** — architecture in arch review, security in security review

---

## STRIDE Threat Modeling

Systematic threat enumeration using Data Flow Diagrams (DFDs).

### DFD Elements

| Element | Symbol | Description |
|---------|--------|-------------|
| External Entity | Rectangle | Users, external systems, APIs |
| Process | Circle/rounded rect | Code that processes data |
| Data Store | Two parallel lines | Databases, files, caches |
| Data Flow | Arrow | Data movement between elements |
| Trust Boundary | Dashed line | Where privilege levels change |

### STRIDE-per-Element Table

| DFD Element | S (Spoofing) | T (Tampering) | R (Repudiation) | I (Info Disclosure) | D (Denial of Service) | E (Elevation of Privilege) |
|-------------|:---:|:---:|:---:|:---:|:---:|:---:|
| External Entity | ✓ | | ✓ | | | |
| Process | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Data Store | | ✓ | ✓ | ✓ | ✓ | |
| Data Flow | | ✓ | | ✓ | ✓ | |

### DREAD Risk Rating

Score each threat 1-10:

| Criterion | 1-3 (Low) | 4-6 (Medium) | 7-10 (High) |
|-----------|-----------|--------------|-------------|
| **D**amage | Minor data loss | Service degradation | Complete data loss |
| **R**eproducibility | Hard, rare conditions | Sometimes | Always |
| **E**xploitability | Expert required | Advanced user | Any user, browser |
| **A**ffected users | Few/specific | Some users | All users |
| **D**iscoverability | Hard to find | With effort | Obvious |

**Risk score** = (D + R + E + A + D) / 5

| Score | Priority | Action |
|-------|----------|--------|
| 7-10 | Critical | Fix before merge |
| 4-7 | High | Fix within sprint |
| 2-4 | Medium | Track in backlog |
| 1-2 | Low | Accept with documentation |

### STRIDE Checklist
- [ ] DFD created for all new services/features
- [ ] Trust boundaries identified and documented
- [ ] Each DFD element analyzed against STRIDE categories
- [ ] Threats scored using DREAD
- [ ] Critical/High threats have mitigations before merge
- [ ] Residual risks documented and accepted by security owner

## OWASP Threat Modeling

### Asset Classification

| Asset Type | Examples | Classification |
|------------|----------|----------------|
| Data | PII, credentials, financial records | Confidentiality + Integrity |
| Services | Auth, payment, notification | Availability + Integrity |
| Infrastructure | Databases, message queues, caches | Availability |
| Code | Source, build artifacts, configs | Integrity |
| Secrets | API keys, encryption keys | Confidentiality |

### Threat Agents

| Agent Type | Motivation | Capability |
|------------|------------|------------|
| External attacker | Financial gain | Varies — script kiddie to APT |
| Malicious insider | Revenge, financial gain | High — has credentials |
| Compromised service | Lateral movement | Medium — service permissions |
| Supply chain | Backdoor, data exfiltration | High — trusted position |

### Attack Surfaces

| Surface | Risk |
|---------|------|
| API endpoints | Injection, broken auth, SSRF |
| Authentication flows | Credential stuffing, session fixation |
| File uploads | Path traversal, malware upload |
| Message queues | Poison messages, replay attacks |
| Webhooks | Forgery, SSRF |

### OWASP Deliverable
- [ ] Asset inventory with classification labels
- [ ] Trust boundary diagram with all crossings marked
- [ ] Attack surface register with mitigations
- [ ] Gap analysis against OWASP ASVS

## PASTA vs STRIDE vs OWASP Selection

| Methodology | Best For | Effort | When |
|-------------|----------|--------|------|
| STRIDE | Component-level threats | Low-Medium | Every PR touching security boundaries |
| OWASP TM | Full system threat modeling | Medium | New services, major features |
| PASTA | Risk-centric, business-aligned | High | High-risk systems, regulatory requirements |

---

## Secure Code Review Patterns

### Cryptography Review

| Use Case | Approved | Banned | Flag |
|----------|----------|--------|------|
| Symmetric encryption | AES-256-GCM, ChaCha20-Poly1305 | DES, 3DES, RC4, AES-CBC (no HMAC) | Non-AEAD mode |
| Asymmetric encryption | RSA-4096, X25519, Ed25519 | RSA-1024, RSA-2048 (new) | Key size < 2048 |
| Hashing | SHA-256, SHA-3, BLAKE2 | MD5, SHA-1 | Any use of MD5/SHA-1 |
| Key derivation | Argon2id, scrypt, PBKDF2 (>600k) | PBKDF2 (<100k), plain hash | Low iteration count |
| TLS | TLS 1.3 (preferred), TLS 1.2 (min) | TLS 1.0/1.1, SSLv3 | Protocol version config |
| Cert validation | Full chain + hostname verification | Disabled validation | `verify=False` |

**Crypto checklist:**
- [ ] No hardcoded keys, IVs, salts, or passwords
- [ ] IVs/nonces via CSPRNG, never reused
- [ ] Authenticated encryption (AEAD) for all symmetric encryption
- [ ] Key rotation mechanism exists and tested
- [ ] No custom crypto (use established libraries)
- [ ] Constant-time comparison for MACs/tokens (`hmac.compare_digest`)

```python
# BAD: ECB mode reveals patterns
cipher = AES.new(key, AES.MODE_ECB)
# GOOD: GCM mode (authenticated encryption)
cipher = AES.new(key, AES.MODE_GCM)

# BAD: Non-constant-time comparison
if user_mac == expected_mac:  # Timing attack
# GOOD: Constant-time comparison
if hmac.compare_digest(user_mac, expected_mac):

# BAD: Weak KDF
dk = hashlib.pbkdf2_hmac('sha256', password, salt, 1000)
# GOOD: Strong KDF
dk = hashlib.pbkdf2_hmac('sha256', password, salt, 600000)
```

### Authentication Flow Review

**OAuth2/OIDC checklist:**
- [ ] `state` parameter used and validated (CSRF)
- [ ] `nonce` parameter for ID tokens (replay prevention)
- [ ] Token storage: HTTP-only secure cookies (not localStorage)
- [ ] Token refresh: rotation implemented
- [ ] Token validation: signature, issuer, audience, expiry all checked
- [ ] Redirect URIs: exact match, no wildcards
- [ ] PKCE used for public clients
- [ ] No tokens in URL query parameters

**Session management:**
- [ ] Session ID via CSPRNG (≥128 bits entropy)
- [ ] Session ID regenerated on login (prevent fixation)
- [ ] Timeout: idle (30 min) and absolute (12h)
- [ ] Invalidated on logout (server-side deletion)
- [ ] `Secure`, `HttpOnly`, `SameSite=Lax` flags

**Password handling:**
- [ ] Hashed with Argon2id (or scrypt/PBKDF2 high cost)
- [ ] No length limit < 72 characters
- [ ] Common password check (HaveIBeenPwned)
- [ ] Rate limiting on login (per account + per IP)
- [ ] Account lockout with exponential backoff
- [ ] No user enumeration (same error for invalid user vs wrong password)
- [ ] MFA enforced for privileged accounts
- [ ] Reset tokens: single-use, time-limited (15 min)

### Input Validation

| Layer | Location | Purpose | Technology |
|-------|----------|---------|------------|
| L1: Transport | Edge/WAF | Block known patterns | ModSecurity, AWS WAF |
| L2: Schema | API Gateway | Structural validation | JSON Schema, OpenAPI |
| L3: Business | Application | Semantic validation | Custom validators |
| L4: Data | Database | Integrity constraints | CHECK constraints |

**Input validation checklist:**
- [ ] All user input validated at L2 (schema) and L3 (business)
- [ ] Allowlist preferred over blocklist
- [ ] Type coercion attacks prevented
- [ ] String inputs: max length enforced, UTF-8 verified
- [ ] File uploads: content-type via magic bytes, size limited, outside webroot
- [ ] URL inputs: scheme allowlist (https only), no `file://`/`gopher://`
- [ ] No ReDoS patterns (catastrophic backtracking)

```python
# SQL injection prevention
# BAD
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
# GOOD
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# SSRF prevention
def validate_url(url):
    parsed = urllib.parse.urlparse(url)
    if parsed.scheme not in ('https',):
        raise ValueError("Only HTTPS allowed")
    ip = socket.getaddrinfo(parsed.hostname, None)[0][4][0]
    if ipaddress.ip_address(ip).is_private:
        raise ValueError("No private IPs allowed")
    return url
```

```javascript
// XSS prevention
// BAD
element.innerHTML = userInput;
// GOOD
element.textContent = userInput;

// Prototype pollution prevention
// BAD
Object.assign(target, userInput);
// GOOD
const sanitized = JSON.parse(JSON.stringify(userInput));
Object.assign(target, sanitized);
```

---

## Compliance-Aware Code Review

### SOC 2 Type II

| Criteria | Control | Code Review Check |
|----------|---------|-------------------|
| CC6.1 | Logical access | Auth required on all endpoints, RBAC enforced |
| CC6.3 | Access removal | Deprovisioning automated, access revoked on role change |
| CC6.7 | Data transmission | TLS 1.2+ on all connections, mTLS internally |
| CC7.1 | Vulnerability management | Dependency scanning, patching SLAs |
| CC7.2 | Monitoring | Security event logging, SIEM integration |
| CC8.1 | Change management | PR reviews required, CI/CD gates |

**SOC 2 checklist:**
- [ ] All access changes logged with actor, target, timestamp
- [ ] Audit logs immutable (append-only, no delete API)
- [ ] Encryption at rest (AES-256) and in transit (TLS 1.2+)
- [ ] Change management: all changes through PR, no direct deploys
- [ ] Separation of duties: deployer ≠ code author
- [ ] Vulnerability scan results gated in CI

### GDPR

| Right | Article | Endpoint | Review Check |
|-------|---------|----------|--------------|
| Erasure | Art. 17 | `DELETE /users/{id}/personal-data` | Cascades to all stores, < 30 days |
| Access | Art. 15 | `GET /users/{id}/data-export` | All PII, machine-readable |
| Rectification | Art. 16 | `PATCH /users/{id}/personal-data` | Propagates downstream |
| Portability | Art. 20 | `GET /users/{id}/data-portability` | JSON/CSV format |
| Restrict | Art. 18 | `POST /users/{id}/restrict-processing` | PII processing stops |
| Object | Art. 21 | `POST /users/{id}/object-processing` | Halt profiling/marketing |

```python
# GDPR — Data erasure pattern
class UserDataService:
    def erase_user_data(self, user_id: str) -> ErasureResult:
        self._verify_data_subject(user_id)
        self.db.delete_user(user_id)
        # Cascade to all downstream stores
        self.search_index.remove_user(user_id)
        self.cache.invalidate(user_id)
        self.analytics.anonymize_user(user_id)
        self.email_service.remove_contact(user_id)
        self.third_party_processors.request_deletion(user_id)
        self.audit_log.record(action="data_erasure", subject_hash=hash(user_id))
        return ErasureResult(status="complete",
            systems_erased=self._get_erasure_manifest(user_id),
            retention_exceptions=self._get_legal_holds(user_id))
```

**GDPR checklist:**
- [ ] Personal data identified and tagged (PII fields annotated)
- [ ] Data minimization: only collect necessary fields
- [ ] Consent recorded with timestamp, scope, version
- [ ] Consent withdrawal as easy as giving consent
- [ ] Data retention automated (auto-delete after period)
- [ ] No PII in logs, error messages, or analytics events
- [ ] DPIA exists for high-risk processing

### HIPAA

| Rule | Requirement | Check |
|------|-------------|-------|
| Access Control (§164.312(a)) | Unique user ID, emergency access, auto-logoff, encryption | All 4 implemented |
| Audit Controls (§164.312(b)) | Log all PHI access (who, what, when) | Immutable and complete |
| Transmission Security (§164.312(e)) | Encrypt PHI in transit | TLS 1.2+ mandatory |
| Encryption at Rest | Encrypt PHI | AES-256, KMS managed |

```python
# HIPAA — PHI access logging
def phi_access_logged(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        current_user = get_authenticated_user()
        result = func(*args, **kwargs)
        audit_logger.info("phi_access", extra={
            "user_id": current_user.id,
            "function": func.__name__,
            "timestamp": datetime.utcnow().isoformat(),
            "fields_accessed": get_phi_field_names(func),
            "patient_hash": hashlib.sha256(
                kwargs.get("patient_id", "").encode()
            ).hexdigest()[:16],
        })
        return result
    return wrapper
```

**HIPAA checklist:**
- [ ] All PHI fields identified and marked
- [ ] PHI access logged: user ID, timestamp, fields, purpose
- [ ] PHI never in URLs, query strings, or error messages
- [ ] PHI encrypted at rest (AES-256) and in transit (TLS 1.2+)
- [ ] Audit logs immutable, retained 6+ years
- [ ] Emergency access (break-glass) with logging
- [ ] Auto-logoff (15 min idle timeout)
- [ ] BAA verified for all third-party services processing PHI

### Compliance Selection Matrix

| Standard | Applies When | Key Focus |
|----------|-------------|-----------|
| SOC 2 | SaaS/B2B, enterprise sales | Access controls, change management |
| GDPR | EU personal data | Data subject rights, consent, erasure |
| HIPAA | US healthcare (PHI) | PHI access controls, audit logging |
| PCI DSS | Payment card data | Cardholder isolation, encryption |
| ISO 27001 | Info security management | ISMS controls, risk assessment |

---

## AI/ML Model Review

### Bias Testing

| Metric | Definition | Threshold |
|--------|-----------|-----------|
| Demographic Parity | Equal prediction rates across groups | < 10% difference |
| Equalized Odds | Equal TPR/FPR across groups | < 5% difference |
| Disparate Impact | 4/5ths rule | Ratio > 0.8 |

**Bias checklist:**
- [ ] Protected attributes identified
- [ ] Training data representativeness audited
- [ ] Fairness metrics computed across protected groups
- [ ] Proxy features identified (zip code → race correlation)
- [ ] Model cards published
- [ ] Ongoing bias monitoring in production
- [ ] Human-in-the-loop for high-stakes decisions

### Prompt Injection Defense

| Vector | Attack | Mitigation |
|--------|--------|------------|
| Direct injection | User overrides system prompt | Input/output separation, delimiters |
| Indirect injection | Malicious content in RAG docs | Content sanitization, trust boundaries |
| Jailbreaking | System prompt extraction | Prompt hardening, output filtering |

```python
def safe_llm_call(user_input: str, system_prompt: str) -> str:
    sanitized = sanitize_user_input(user_input)
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": f"<user_input>\n{sanitized}\n</user_input>"},
    ]
    response = llm.chat(messages=messages, max_tokens=1000, temperature=0.0)
    filtered = filter_pii(response.content)
    filtered = filter_system_prompt_leakage(filtered, system_prompt)
    return filtered
```

**Prompt injection checklist:**
- [ ] User input never directly concatenated with system prompt
- [ ] Delimiters/encodings separate instructions from data
- [ ] Output filtering: no PII leakage, no system prompt disclosure
- [ ] Tool/function calls validated before execution
- [ ] RAG source trust boundaries enforced
- [ ] Rate limiting on LLM endpoints
- [ ] Red team testing for jailbreaks

---

## Performance Review Patterns

### Complexity Thresholds

| Context | Max Acceptable | Flag |
|---------|---------------|------|
| Hot path (API handler) | O(n log n) | O(n²) or worse |
| Background job | O(n²) acceptable | O(n³) or worse |
| Database query | O(n log n) with index | Full table scan >10k rows |

```python
# BAD: O(n²) — nested loop
for i, a in enumerate(items):
    for j, b in enumerate(items):
        if i != j and a == b: ...
# GOOD: O(n) — hash set
seen = set()
for item in items:
    if item in seen: ...
    seen.add(item)
```

**Complexity checklist:**
- [ ] Hot-path functions analyzed for time complexity
- [ ] No O(n²) on user-facing endpoints without justification
- [ ] Database queries use appropriate indexes
- [ ] N+1 queries eliminated
- [ ] Pagination on all list endpoints

### Memory Leak Patterns

```python
# BAD: Unbounded cache
cache = {}
def get_user(user_id):
    if user_id not in cache:
        cache[user_id] = db.fetch(user_id)
    return cache[user_id]

# GOOD: LRU cache
@lru_cache(maxsize=1000)
def get_user(user_id):
    return db.fetch(user_id)
```

**Memory checklist:**
- [ ] All resources use context managers / try-finally / defer
- [ ] Caches have bounded size (LRU, TTL)
- [ ] Event listeners cleaned up on teardown
- [ ] No global mutable state that grows unboundedly

### Concurrency Bugs

```python
# BAD: Race condition
if not os.path.exists(path):
    with open(path, 'w') as f: f.write(data)
# GOOD: Atomic create
fd = os.open(path, os.O_CREAT | os.O_EXCL | os.O_WRONLY)

# BAD: Non-atomic read-modify-write
counter = get_counter(); counter += 1; set_counter(counter)
# GOOD: Atomic increment
db.execute("UPDATE counters SET value = value + 1 WHERE id = %s", (id,))
```

**Concurrency checklist:**
- [ ] Shared mutable state protected by locks/atomics/immutability
- [ ] Lock ordering consistent (prevent deadlocks)
- [ ] No TOCTOU vulnerabilities
- [ ] Database transactions use appropriate isolation level
- [ ] Timeout on all external calls
- [ ] Idempotency keys for retried operations

---

## OWASP API Security Top 10 2023

| # | Name | Attack Vector | Fix |
|---|------|--------------|-----|
| API1 | Broken Object Level Auth (BOLA) | Manipulate object IDs | Per-object ownership checks |
| API2 | Broken Authentication | Credential stuffing, token abuse | Rate limiting, short-lived tokens, MFA |
| API3 | Broken Object Property Auth | Mass assignment, excessive data | Explicit allowlist DTOs |
| API4 | Unrestricted Resource Consumption | No rate limits | Rate limiting, pagination caps |
| API5 | Function Level Authorization | Guess admin endpoints | RBAC on every endpoint |
| API6 | Sensitive Business Flows | Bot abuse | CAPTCHA, behavioral analysis |
| API7 | SSRF | Internal service access | URL allowlist, block private IPs |
| API8 | Security Misconfiguration | Verbose errors, CORS | Security headers, strict CORS |
| API9 | Improper Inventory | Shadow APIs | API inventory, deprecate old versions |
| API10 | Unsafe API Consumption | Trusting third-party data | Validate all external responses |

### BOLA Mitigation Pattern

```python
# BAD: No ownership check
@app.get("/orders/{order_id}")
def get_order(order_id: str):
    return db.get_order(order_id)

# GOOD: Ownership check via authenticated context
@app.get("/orders/{order_id}")
def get_order(order_id: str, user = Depends(get_current_user)):
    order = db.get_order(order_id)
    if order.owner_id != user.id and user.role != "admin":
        raise HTTPException(status_code=404)  # 404, not 403
    return order
```

### Mass Assignment Prevention

```python
# BAD: User can set any field including role
@app.put("/users/me")
def update_user(data: dict, user = Depends(get_current_user)):
    db.update_user(user.id, data)

# GOOD: Explicit allowlist DTO
class UserUpdateRequest(BaseModel):
    name: str | None = None
    email: str | None = None
    # role is NOT included

@app.put("/users/me")
def update_user(data: UserUpdateRequest, user = Depends(get_current_user)):
    db.update_user(user.id, data.model_dump(exclude_unset=True))
```

### SSRF Prevention

```python
BLOCKED_CIDRS = [
    "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
    "127.0.0.0/8", "169.254.0.0/16", "::1/128", "fc00::/7",
]

def validate_url(url: str) -> str:
    parsed = urlparse(url)
    if parsed.scheme not in ("https",):
        raise ValueError("Only HTTPS allowed")
    ip = socket.getaddrinfo(parsed.hostname, None)[0][4][0]
    addr = ipaddress.ip_address(ip)
    for cidr in BLOCKED_CIDRS:
        if addr in ipaddress.ip_network(cidr):
            raise ValueError(f"Blocked: {ip} in {cidr}")
    return url
```

---

## Supply Chain Security

### SLSA Levels

| Level | Build | Provenance | Key Guarantee |
|-------|-------|------------|---------------|
| SLSA 0 | No requirements | None | No guarantees |
| SLSA 1 | Scripted build | Unsigned provenance | Provenance available |
| SLSA 2 | Hosted CI platform | Signed by build service | Tamper resistance |
| SLSA 3 | Hardened, isolated, ephemeral | Non-falsifiable signature | Tamper proof |

### Supply Chain Attack Case Studies

**SolarWinds (2020):** Compromised build system injected backdoor. 18,000+ orgs affected.
- Build system must be isolated from source repo access
- Build process must be reproducible
- Multiple independent build pipelines for critical artifacts

**npm attacks:** `event-stream` (2018), `ua-parser-js` (2021), `colors`/`faker` (2022).
- Lockfile committed and verified in CI
- Dependencies pinned by hash
- `npm audit` run in CI with failure on high/critical
- No `preinstall`/`postinstall` from untrusted packages

**PyPI attacks:** `colorama` typosquat (2022), dependency confusion, `ctx` backdoor (2022).
- Internal packages scoped to private registry
- `pip-audit` run in CI
- Hash checking mode (`--require-hashes`)

### Dependency Verification Checklist
- [ ] Lockfile entries with integrity hashes
- [ ] SBOM generated for every build (Syft, SPDX, CycloneDX)
- [ ] Dependency vulnerability scanning in CI (Grype, Trivy, Snyk)
- [ ] New dependency additions require security review
- [ ] No post-install scripts from dependencies (or allowlisted)
- [ ] Private registry prevents dependency confusion
- [ ] Base container images pinned by digest, not tag

```bash
# Generate and scan SBOM
syft dir:. -o spdx-json > sbom.spdx.json
grype sbom:sbom.spdx.json --fail-on high

# npm: verify lockfile integrity
npm ci --ignore-scripts
npm audit --audit-level=high
```

---

## LLM/AI Security (OWASP LLM Top 10)

| Rank | Vulnerability | Review Focus |
|------|--------------|--------------|
| LLM01 | Prompt Injection | Input separation, output filtering |
| LLM02 | Sensitive Info Disclosure | Output filtering, PII detection |
| LLM03 | Supply Chain | Model provenance, data validation |
| LLM04 | Data/Model Poisoning | Data provenance, anomaly detection |
| LLM05 | Improper Output Handling | Output encoding, sandboxing |
| LLM06 | Excessive Agency | Least privilege, human approval |
| LLM07 | System Prompt Leakage | Prompt hardening, output filtering |
| LLM08 | Vector/Embedding Weaknesses | Source trust boundaries |
| LLM09 | Misinformation | Fact-checking, source attribution |
| LLM10 | Unbounded Consumption | Rate limiting, token budgets |

### Excessive Agency Defense

```python
FUNCTION_PERMISSIONS = {
    "read_user_profile": {"risk": "low", "confirmation": False},
    "update_user_email": {"risk": "medium", "confirmation": True},
    "delete_user_account": {"risk": "critical", "confirmation": True, "mfa_required": True},
    "execute_sql": {"risk": "critical", "disabled": True},
}

async def execute_llm_function(function_name: str, args: dict, user: User):
    perms = FUNCTION_PERMISSIONS.get(function_name, {"disabled": True})
    if perms.get("disabled"):
        raise PermissionError(f"Function {function_name} is disabled")
    if perms.get("confirmation"):
        return PendingConfirmation(function=function_name, args=args)
    return await call_function(function_name, args)
```

---

## Container Security Scanning

### Scanner Comparison

| Feature | Trivy | Grype | Snyk Container |
|---------|-------|-------|----------------|
| License | Apache 2.0 | Apache 2.0 | Commercial (free tier) |
| Speed | Fast | Fast | Medium |
| Dockerfile lint | Yes | No | Yes |
| Secret scanning | Yes | No | Yes |
| IaC scanning | Yes | No | No |
| Fix guidance | Yes | No | Yes |
| Best for | All-in-one | Fast dep scanning | Developer workflow |

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest
grype myapp:latest --fail-on high
snyk container test myapp:latest --severity-threshold=high
```

### Dockerfile Checklist
- [ ] Non-root user (`USER` directive)
- [ ] Base image pinned by digest, not `latest`
- [ ] Multi-stage build
- [ ] No secrets in build args or ENV
- [ ] `.dockerignore` excludes `.env`, `.git`, `node_modules`
- [ ] `HEALTHCHECK` defined
- [ ] Minimal base image (distroless, alpine, scratch)

```bash
trivy config Dockerfile
checkov -f Dockerfile
hadolint Dockerfile
```

---

## IaC Security Scanning

### Scanner Comparison

| Feature | Checkov | tfsec | KICS |
|---------|---------|-------|------|
| Terraform | Excellent | Excellent (dedicated) | Good |
| CloudFormation | Yes | No | Yes |
| Kubernetes | Yes | Limited | Yes |
| Custom rules | Python/Bicep | Go | Rego (OPA) |
| Best for | Multi-platform | Terraform-specific | Multi-platform + OPA |

```bash
checkov -d . --framework terraform,kubernetes,cloudformation
tfsec . --minimum-severity HIGH
kics scan -p . --fail-on high
```

### Common IaC Misconfigurations

```hcl
# BAD: Public S3 bucket
resource "aws_s3_bucket_acl" "data" {
  acl = "public-read"  # CRITICAL
}

# GOOD: Private with encryption
resource "aws_s3_bucket_public_access_block" "data" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

```yaml
# BAD: Privileged container
securityContext:
  privileged: true
  runAsRoot: true

# GOOD: Restricted
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

**IaC checklist:**
- [ ] No public S3 buckets/databases without justification
- [ ] Encryption at rest for all storage
- [ ] Security groups: no `0.0.0.0/0` on non-HTTP ports
- [ ] IAM policies follow least privilege
- [ ] K8s: no privileged containers, resource limits set
- [ ] State file encrypted and access-controlled

---

## Secret Detection

### Scanner Comparison

| Feature | git-secrets | TruffleHog | detect-secrets |
|---------|------------|------------|----------------|
| Approach | Pattern (regex) | Pattern + verified | Pattern (plugins) |
| Git integration | Hooks (pre-commit) | Full history scan | Pre-commit hook |
| Verified scanning | No | Yes (checks if live) | No |
| Best for | AWS repos | Comprehensive | Python repos |

```bash
git secrets --install && git secrets --register-aws
trufflehog git file://. --only-verified
detect-secrets scan > .secrets.baseline
```

**Secret detection checklist:**
- [ ] Secret scanning in CI (blocks merge)
- [ ] Pre-commit hooks installed
- [ ] Full git history scanned at least once
- [ ] Rotation procedure: if secret in history → rotate immediately
- [ ] `.gitignore` includes `.env`, `*.pem`, `*.key`

---

## Policy as Code

### Tool Comparison

| Feature | OPA (Rego) | Kyverno | Sentinel | Cloud Custodian |
|---------|-----------|---------|----------|-----------------|
| Language | Rego | YAML | HCL | YAML |
| Platform | Any | Kubernetes only | HashiCorp | AWS/Azure/GCP |
| Learning curve | High | Low | Medium | Low |
| Mutating webhooks | No | Yes | No | No |
| Best for | Complex multi-platform | K8s admission | Terraform governance | Cloud resource hygiene |

```rego
# OPA — Deny containers not from approved registry
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    not startswith(container.image, "registry.example.com/")
    msg := sprintf("Container '%s' must use approved registry", [container.name])
}
```

```yaml
# Kyverno — Require resource limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-container-resources
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "All containers must have CPU and memory limits"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

**Policy as Code checklist:**
- [ ] Policies version-controlled with infrastructure code
- [ ] Policies tested before deployment
- [ ] Policy violations block PR merge
- [ ] Admission controller enforces at deploy time
- [ ] Policy exceptions documented with approval and expiration
- [ ] Compliance mapping: each policy → regulatory control

---

## Zero Trust Architecture

### NIST SP 800-207 Core Tenets

1. All data sources and computing services are resources
2. All communication secured regardless of network location
3. Access granted per-session, not persistent
4. Access determined by dynamic policy (identity, device, context)
5. Enterprise monitors integrity/security posture of all assets
6. Authentication and authorization dynamic and strictly enforced
7. Enterprise collects maximum info for trust decisions

### Zero Trust Deployment Models

| Model | Description | Complexity |
|-------|-------------|------------|
| Network-centric | Micro-segmentation, SDN/SDP | Medium |
| Identity-centric | Strong identity + MFA + device trust | Medium |
| Application-centric | Per-app access proxy (BeyondCorp) | High |
| Data-centric | Classification-driven, DLP + encryption | High |

**Zero Trust checklist:**
- [ ] No implicit trust based on network location
- [ ] All inter-service calls authenticated (mTLS, JWT)
- [ ] All inter-service calls authorized (policy engine, RBAC)
- [ ] Lateral movement prevented by network policies
- [ ] MFA enforced for all human access
- [ ] Policy engine: deny by default if unavailable (fail closed)

### SPIFFE/SPIRE Service Identity

**SPIFFE ID:** `spiffe://trust-domain/workload-identifier`

| SVID Type | Lifetime | Use Case |
|-----------|----------|----------|
| X.509 SVID | Hours | mTLS between services |
| JWT SVID | Minutes | API auth, user delegation |

**SPIFFE/SPIRE checklist:**
- [ ] SPIFFE IDs follow naming convention (hierarchical, stable)
- [ ] Short SVID lifetimes (X.509: 1h, JWT: 15min), auto-rotation
- [ ] No static credentials alongside SPIFFE
- [ ] Node attestation uses cloud-native mechanism (not join tokens in prod)

### mTLS Service Mesh Comparison

| Feature | Istio | Linkerd | Cilium |
|---------|-------|---------|--------|
| Data plane | Envoy (C++) | linkerd2-proxy (Rust) | eBPF + Envoy |
| Cert lifetime | 24h, configurable | 24h, configurable | Configurable |
| Auto-mTLS | `PeerAuthentication: STRICT` | On by default | `encryption.enabled: true` |
| Policy | AuthorizationPolicy (L4/L7) | ServerAuthorization | CiliumNetworkPolicy (L3-L7) |
| Resource overhead | High (~50MB/pod) | Low (~10MB/pod) | Lowest (eBPF) |
| Best for | Complex policies | Simplicity | Performance-critical |

```yaml
# Istio strict mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

**mTLS checklist:**
- [ ] STRICT mode in production (no PERISSIVE except migration)
- [ ] Certificate lifetime ≤ 24h
- [ ] AuthorizationPolicy applied (mTLS alone ≠ authorization)
- [ ] Fallback behavior: deny by default if CA unavailable

---

## Secret Management

### Comparison

| Feature | Vault | AWS Secrets Manager | Sealed Secrets | External Secrets Operator |
|---------|-------|--------------------|----|----|
| Dynamic secrets | Yes | No (rotation only) | No | No (syncs) |
| Multi-cloud | Yes | AWS only | Any K8s | Any K8s + any provider |
| GitOps safe | No | No | Yes | Yes |
| Complexity | High | Low | Low | Medium |
| Best for | Dynamic creds, PKI | AWS-native | Simple GitOps | Multi-provider sync |

```yaml
# Vault Agent Injector
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "my-app"
  vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/my-role"
```

**Secret management checklist:**
- [ ] No secrets in code, config files, or container images
- [ ] No secrets in env vars visible in pod spec (use volume mount)
- [ ] Dynamic secrets preferred (Vault database engine)
- [ ] Secret TTL ≤ 24h for dynamic creds
- [ ] Secret access audited (who, what, when)
- [ ] Emergency access procedure documented

---

## Container Runtime Security

### Runtime Security Comparison

| Feature | Falco | Sysdig Secure | KubeArmor |
|---------|-------|---------------|-----------|
| Primary function | Detection (alert) | Detection + Response | Enforcement (block) |
| Kernel integration | eBPF / module | eBPF | AppArmor / BPF-LSM / SELinux |
| Enforcement | No (alert only) | Yes (kill, pause, isolate) | Yes (block at kernel) |
| Overhead | Low (2-5% CPU) | Medium (3-7%) | Low (1-3% BPF-LSM) |
| License | Apache 2.0 | Commercial | Apache 2.0 |
| Best for | Detection + audit | Full platform | Active blocking |

```yaml
# Falco rule — shell in container
- rule: Shell in Container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, zsh, dash, ksh)
  output: "Shell spawned (user=%user.name container=%container.name shell=%proc.name)"
  priority: WARNING
```

```yaml
# KubeArmor — block shell in webapp
apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: block-shell-in-webapp
spec:
  selector:
    matchLabels:
      app: webapp
  process:
    matchPaths:
      - path: /bin/sh
        action: Block
      - path: /bin/bash
        action: Block
```

**Runtime security checklist:**
- [ ] Runtime detection deployed (Falco or equivalent)
- [ ] Custom rules for org-specific threats
- [ ] Alert routing: CRITICAL → PagerDuty, WARNING → SIEM
- [ ] Enforcement for sensitive workloads (KubeArmor/Sysdig)
- [ ] Alert noise tuning: false positive rate < 5%

---

## DevSecOps Pipeline Security Gates

| Phase | Gate | Tools | FAIL Criteria |
|-------|------|-------|---------------|
| Plan | Threat Model | STRIDE, OWASP Threat Dragon | Unmitigated HIGH threats |
| Code | Pre-commit | detect-secrets, trufflehog | Any secret in diff |
| Build | SCA + SAST | Trivy, Semgrep, CodeQL | CRITICAL CVE; HIGH+ SAST |
| Test | DAST + IAST | ZAP, Burp Suite | OWASP Top 10 confirmed |
| Deploy | IaC Scan | Checkov, tfsec | Public S3; unencrypted DB; root |
| Operate | Runtime | Falco, Sysdig | Crypto mining; anomalous exec |

**FAIL BUILD summary:**
```
NEVER FAIL: INFO severity, low CVEs with no exploit, dev-only dep vulns
ALWAYS FAIL: Secrets in code, CRITICAL CVE with exploit, OWASP Top 10 confirmed,
             public storage with sensitive data, root container in prod manifest,
             unencrypted PII in IaC, zero-day (EPSS > 0.7)
CONDITIONAL: HIGH without exploit, MEDIUM SAST in non-critical, deprecated API usage
```

---

## OWASP SAMM Maturity Assessment

### 5 Business Functions, 15 Practices

| Function | Practices |
|----------|-----------|
| Governance | Strategy & Metrics, Policy & Compliance, Education & Guidance |
| Design | Threat Assessment, Security Requirements, Security Architecture |
| Implementation | Secure Build, Secure Deployment, Defect Management |
| Verification | Architecture Assessment, Requirements Testing, Security Testing |
| Operations | Incident Management, Environment Management, Operational Management |

### Maturity Levels

| Level | Name | Description |
|-------|------|-------------|
| 0 | Implicit | No formal practice |
| 1 | Initial | Basic practices in place |
| 2 | Defined | Documented and standardized |
| 3 | Managed | Measured and controlled |
| 4 | Optimized | Continuous improvement |

**SAMM checklist:**
- [ ] Assessment completed for 5 functions, 15 practices scored 0-3
- [ ] Maturity gaps identified between current and target
- [ ] Improvement roadmap created
- [ ] Re-assessment scheduled (quarterly)

---

## CSPM (Cloud Security Posture Management)

### CSPM vs CWPP vs CNAPP

| Feature | CSPM | CWPP | CNAPP |
|---------|------|------|-------|
| Focus | Infrastructure misconfig | Runtime workload protection | Unified |
| Scope | Cloud control plane | VMs, containers, serverless | Full cloud-native |
| Detection | Config drift, compliance | Malware, exploits | Both + shift-left |
| Agent | API-based (agentless) | Agent-based | Both |
| Examples | AWS Config, Prisma Cloud | Aqua, Sysdig | Wiz, Orca |

### Critical Cloud Misconfigurations

| Misconfig | Risk | Fix |
|-----------|------|-----|
| Public S3 bucket | Data exfiltration | Block public access |
| Open Security Group (0.0.0.0/0) | Lateral movement | Restrict to known CIDRs |
| Unencrypted RDS/EBS | Data breach at rest | Enable KMS encryption |
| Over-permissive IAM (*:*) | Privilege escalation | Least privilege |
| Public IP on DB | Direct DB access | Set false, use bastion |

---

## API Security Testing

### Burp Suite BApps for API Testing
```
Autorize — Authorization testing (BOLA/BFLA)
JSON Web Tokens — JWT manipulation
Active Scan++ — Enhanced scanning
Param Miner — Hidden parameter discovery
InQL — GraphQL testing
Turbo Intruder — High-speed fuzzing
```

### ZAP CI/CD Integration
```yaml
# GitHub Actions — ZAP API scan
api-security:
  runs-on: ubuntu-latest
  steps:
    - name: ZAP API Scan
      uses: zaproxy/action-api-scan@v0.7.0
      with:
        target: https://staging.example.com/openapi.json
        format: openapi
```

---

## Sigstore & SBOM

### Cosign Usage
```bash
# Sign (keyless — uses OIDC)
cosign sign --yes ghcr.io/org/app@sha256:abc123

# Verify
cosign verify \
  --certificate-identity=user@example.com \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  ghcr.io/org/app@sha256:abc123

# Attach SBOM
cosign attest --predicate sbom.spdx.json --type spdxjson \
  ghcr.io/org/app@sha256:abc123
```

### SBOM Tools

| Tool | Format | Strengths |
|------|--------|-----------|
| Syft | SPDX, CycloneDX | Fast, accurate, multiple formats |
| Grype | Reads SPDX/CycloneDX | Fast vuln matching, EPSS scoring |
| Trivy | SPDX, CycloneDX | All-in-one (SBOM + vuln + secret) |

```bash
syft dir:. -o spdx-json > sbom.spdx.json
grype sbom:sbom.spdx.json --fail-on high
trivy image --format cyclonedx --output sbom.cdx.json ghcr.io/org/app:v1.0
```

**Supply chain security final checklist:**
- [ ] SLSA level assessed (target: L2 minimum)
- [ ] All container images signed (Sigstore/Cosign)
- [ ] SBOM generated at build time
- [ ] Dependency pins use hash, not tags
- [ ] No unpinned GitHub Actions
- [ ] Provenance attestation stored (Rekor)
- [ ] Signature verification in deployment pipeline
- [ ] Renovate/Dependabot for dependency updates

---

## Microservices Security Architecture

### Zero Trust Principles

| Principle | Implementation | Review Check |
|-----------|---------------|--------------|
| Verify explicitly | Auth every request with strong identity | No implicit trust from network location |
| Least privilege | JIT/JEA access | Services have minimum required permissions |
| Assume breach | Encrypt all, segment networks | Blast radius minimized by design |

### Service Identity & mTLS

| Pattern | Mechanism | Trade-off |
|---------|-----------|-----------|
| JWT propagation | Pass JWT through chain | Stateless, token size grows |
| OAuth2 token exchange | Exchange at each hop (RFC 8693) | Secure, adds latency |
| SPIFFE/SPIRE | Workload identity separate from user | Strong, infrastructure overhead |
| OPA/Gatekeeper | Policy eval at each hop | Flexible, management complexity |

**Identity propagation checklist:**
- [ ] User identity propagated to all downstream services
- [ ] Service-to-service uses workload identity (not forwarded user tokens)
- [ ] Token validation at every hop
- [ ] No hardcoded service credentials
- [ ] Token audience restriction prevents replay across services

### Network Segmentation

| Layer | Tool | Review Check |
|-------|------|--------------|
| K8s NetworkPolicy | Calico, Cilium | Default-deny ingress; explicit allow per service |
| Service mesh authz | Istio AuthorizationPolicy | L7 policies (method, path, headers) |
| Cloud VPC/subnet | AWS VPC, GCP VPC | Separate subnets, no unnecessary peering |
| API Gateway | Kong, Envoy | Rate limiting, auth, WAF at edge |
| Egress controls | Istio ServiceEntry | Only reach required external endpoints |

**Network segmentation checklist:**
- [ ] Default-deny network policies in all namespaces
- [ ] Explicit allow rules per service-to-service
- [ ] L7 authorization policies (not just L3/L4)
- [ ] Egress restricted to known external dependencies
- [ ] No hostNetwork/hostPID on production pods
- [ ] Database/cache not directly accessible from outside cluster
- [ ] Admin interfaces on separate network segment

### Microservices Security Master Checklist

**Authentication & Authorization:**
- [ ] Centralized identity provider (Keycloak, Auth0, Cognito)
- [ ] OAuth2/OIDC for user auth (no custom auth)
- [ ] Service-to-service via mTLS + SPIFFE or short-lived JWT
- [ ] RBAC/ABAC at service level (not just gateway)
- [ ] Token validation at every service (don't trust gateway alone)

**Data Protection:**
- [ ] Encryption at rest (AES-256, cloud KMS)
- [ ] Encryption in transit (TLS 1.3, mTLS internally)
- [ ] PII field-level encryption or tokenization
- [ ] Data classification labels on all data stores
- [ ] No sensitive data in logs (redact PII, credentials)
- [ ] Data retention policies enforced automatically

**Observability & Incident Response:**
- [ ] Structured logging with correlation IDs
- [ ] Security event logging (auth failures, access denials)
- [ ] Alerting on anomalous patterns (spike in 401/403)
- [ ] Distributed tracing (Jaeger, Zipkin, OpenTelemetry)
- [ ] Runbooks for common security incidents

**Resilience:**
- [ ] Circuit breakers between services
- [ ] Bulkhead pattern (isolate failure domains)
- [ ] Graceful degradation (fallback, not 500 errors)
- [ ] DDoS protection at edge (CDN, WAF)
- [ ] Backup/recovery tested (RTO/RPO documented)

**Deployment & Configuration:**
- [ ] Immutable infrastructure (no SSH into containers)
- [ ] Config from ConfigMaps/Secrets (not baked into images)
- [ ] Security context: non-root, read-only FS, drop all capabilities
- [ ] Pod security standards enforced (restricted profile)
- [ ] Canary/blue-green with auto-rollback on security regression

---

## Advanced Threat Modeling

### Attack Trees

Hierarchical decomposition of attack goals. Root = attacker objective, leaves = exploit steps.

```
[Root Goal: Steal user data]
├── [AND] Compromise auth layer
│   ├── [OR] Credential stuffing
│   │   ├── Leaked credential databases
│   │   └── Password spray attacks
│   └── [OR] Session hijacking
│       ├── XSS-based cookie theft
│       └── Session fixation
├── [AND] Access data store
│   ├── [OR] SQL injection
│   └── [OR] SSRF to internal DB
└── [AND] Exfiltrate data
    ├── [OR] DNS tunneling
    └── [OR] Encrypted channel to C2
```

**Attack tree workflow:**
1. Identify root goal from PR context
2. Build tree with AND (all required) / OR (any sufficient) gates
3. For each leaf, check if PR introduces or mitigates that vector
4. Calculate attack cost: sum effort of cheapest OR-path
5. Flag if attack cost < defense cost

**Attack tree checklist:**
- [ ] Root goal clearly defined
- [ ] All OR-paths evaluated
- [ ] Cheapest attack path identified and mitigated
- [ ] New attack surfaces from PR are leaf nodes
- [ ] Defense cost proportional to attack cost

### Cyber Kill Chain (Lockheed Martin)

| Stage | Description | Code Review Focus | Mitigation |
|-------|-------------|-------------------|------------|
| 1. Reconnaissance | Target research | Rate limiting, info leakage | No verbose errors in prod |
| 2. Weaponization | Crafting exploit | Input validation | Schema validation, allowlisting |
| 3. Delivery | Transmitting exploit | Transport security | TLS 1.3, CSP headers |
| 4. Exploitation | Triggering vuln | Surface reduction | Parameterized queries, sandboxing |
| 5. Installation | Persistence | File integrity | Immutable containers, read-only FS |
| 6. C2 | Remote control | Egress monitoring | Network policies, DNS filtering |
| 7. Actions | Data theft | Data protection | Encryption, DLP, audit logging |

**Kill chain checklist:**
- [ ] Each PR change mapped to relevant kill chain stages
- [ ] Defense at minimum 3 stages (defense in depth)
- [ ] Stage 4 defenses verified for all user inputs
- [ ] Stage 6 egress paths restricted by network policy
- [ ] Stage 7 data protection verified for sensitive assets

### MITRE ATT&CK Mapping

| Tactic ID | Name | Review Focus |
|-----------|------|-------------|
| TA0001 | Initial Access | Phishing, valid accounts, supply chain |
| TA0002 | Execution | Command injection, deserialization |
| TA0003 | Persistence | Backdoors, scheduled tasks |
| TA0004 | Privilege Escalation | SUID, kernel exploits |
| TA0005 | Defense Evasion | Obfuscation, log deletion |
| TA0006 | Credential Access | Brute force, credential dumping |
| TA0008 | Lateral Movement | Remote services, SSH hijacking |
| TA0010 | Exfiltration | Over C2, DNS, alt protocols |
| TA0040 | Impact | Data destruction, ransomware |

**Common technique-to-code mappings:**

| Technique ID | Name | Code Pattern to Review |
|-------------|------|----------------------|
| T1059 | Command Interpreter | `os.system()`, `eval()`, `exec()` |
| T1078 | Valid Accounts | Default creds, weak password policy |
| T1190 | Exploit Public-Facing App | SQLi, XSS, SSRF in endpoints |
| T1053 | Scheduled Task | Cron/K8s CronJobs with excessive perms |
| T1071 | App Layer Protocol | DNS tunneling, HTTP beaconing |
| T1486 | Data Encrypted for Impact | Ransomware, backup integrity |

**MITRE ATT&CK checklist:**
- [ ] All user-facing endpoints mapped to T1190 mitigations
- [ ] No eval/exec/system calls (T1059) without sandboxing
- [ ] No default/weak credentials (T1078)
- [ ] Scheduled tasks run with least privilege (T1053)
- [ ] Egress monitoring covers T1048/T1071 exfil vectors
- [ ] Backup integrity verified against T1486

```bash
# Threat modeling automation
pip install pytm
pytm --dfd diagram ./model.py  # Generate DFDs from annotated Python

pip install threatspec
threatspec init && threatspec report  # Threat model from source annotations
```

---

## ISO 27001 for Engineering

### ISMS Clauses (4–10)

| Clause | Requirement | Engineering Mapping |
|--------|------------|---------------------|
| 4 | Context | Define in-scope systems, data classification |
| 5 | Leadership | Security policy in repo, CISO sign-off |
| 6 | Planning | Threat model per service, risk register |
| 7 | Support | Security training, tool budgets |
| 8 | Operation | CI/CD security gates, SAST/DAST |
| 9 | Performance | Security metrics, quarterly internal audits |
| 10 | Improvement | Post-incident reviews, maturity progression |

### Annex A Engineering Controls (A.8.25–A.8.34)

| Control | Name | Implementation |
|---------|------|----------------|
| A.8.25 | Secure dev lifecycle | SAST/DAST in CI, code review required |
| A.8.26 | App security requirements | Security requirements in user stories |
| A.8.27 | Secure architecture | Architecture review, defense-in-depth |
| A.8.28 | Secure coding | Coding standards (CWE Top 25), input validation |
| A.8.29 | Security testing | SAST (Semgrep), DAST (ZAP), SCA (Trivy) |
| A.8.30 | Outsourced dev | Vendor security assessment, code ownership |
| A.8.31 | Separation of envs | No prod data in dev, different access per env |
| A.8.32 | Change management | PR-based, approval gates, rollback |
| A.8.33 | Test information | No real PII/PHI in tests, synthetic data |
| A.8.34 | Test data protection | Automated scrubbing, retention limits |

```yaml
# CI/CD pipeline mapped to ISO 27001 controls
stages:
  pre-commit:
    controls: [A.8.28, A.8.25]
    checks: [secret-scanning, linting]
  pull-request:
    controls: [A.8.25, A.8.29, A.8.32]
    checks: [code-review (2 approvers), sast, sca]
  build:
    controls: [A.8.27, A.8.29, A.8.31]
    checks: [container-scanning, sbom-generation, image-signing]
  deploy:
    controls: [A.8.31, A.8.32]
    checks: [iac-validation, deployment-approval, rollback-tested]
```

### ISO 27001 Audit Readiness
- [ ] Security policy published and acknowledged
- [ ] Risk register per service (asset → threat → control → residual risk)
- [ ] Threat models for all internet-facing services
- [ ] Access reviews quarterly (evidence + remediation)
- [ ] Vulnerability management: scan → triage → remediate within SLA
- [ ] Change management evidence: PR logs, approvals, deployment logs
- [ ] Incident response documented, tested annually
- [ ] Asset inventory with classification
- [ ] Metrics: mean time to patch, % code reviewed, vuln aging

### Risk Assessment (5×5 Matrix)

```python
from enum import IntEnum

class Likelihood(IntEnum):
    RARE = 1; UNLIKELY = 2; POSSIBLE = 3; LIKELY = 4; ALMOST_CERTAIN = 5

class Impact(IntEnum):
    NEGLIGIBLE = 1; MINOR = 2; MODERATE = 3; MAJOR = 4; CATASTROPHIC = 5

# Risk score = likelihood × impact
# 1-4: low, 5-9: medium, 10-16: high, 17-25: critical
# Treatment: accept, mitigate, transfer, avoid
```

---

## HIPAA Field-Level Encryption

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

class PHIEncryption:
    """AES-256-GCM field-level encryption for PHI."""

    def encrypt_field(self, plaintext: str, field_name: str) -> bytes:
        field_key = self._derive_key(self._get_master_key(), field_name)
        aesgcm = AESGCM(field_key)
        nonce = os.urandom(12)
        ciphertext = aesgcm.encrypt(
            nonce, plaintext.encode("utf-8"),
            field_name.encode("utf-8"),  # AAD
        )
        return nonce + ciphertext

    def decrypt_field(self, data: bytes, field_name: str) -> str:
        nonce, ciphertext = data[:12], data[12:]
        field_key = self._derive_key(self._get_master_key(), field_name)
        return AESGCM(field_key).decrypt(
            nonce, ciphertext, field_name.encode("utf-8")
        ).decode("utf-8")

    def _derive_key(self, master_key: bytes, context: str) -> bytes:
        from cryptography.hazmat.primitives.kdf.hkdf import HKDF
        from cryptography.hazmat.primitives import hashes
        return HKDF(algorithm=hashes.SHA256(), length=32, salt=None,
                     info=f"phi-field-{context}".encode()).derive(master_key)
```

### HIPAA Audit Log (Immutable, 6-Year Retention)

```python
class HIPAAAuditLog:
    """Immutable PHI access audit log with integrity chain."""

    def record_access(self, user_id, action, resource_type, resource_id,
                      fields_accessed, purpose, ip_address, success):
        event = {
            "event_id": generate_ulid(),
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "user_id": user_id,
            "action": action,
            "resource_id_hash": hashlib.sha256(resource_id.encode()).hexdigest()[:16],
            "fields_accessed": fields_accessed,
            "purpose": purpose,
            "ip_address": ip_address,
            "previous_event_hash": self._previous_hash,
        }
        event_hash = hashlib.sha256(json.dumps(event, sort_keys=True).encode()).hexdigest()
        event["event_hash"] = event_hash
        self.storage.append(event)  # WORM storage — no delete
        self._previous_hash = event_hash
        return event_hash

    def verify_chain_integrity(self) -> bool:
        events = self.storage.read_all()
        previous_hash = None
        for event in events:
            if event.get("previous_event_hash") != previous_hash:
                return False  # Chain broken — tampering
            stored_hash = event.pop("event_hash")
            computed = hashlib.sha256(json.dumps(event, sort_keys=True).encode()).hexdigest()
            if computed != stored_hash:
                return False  # Event modified
            previous_hash = stored_hash
            event["event_hash"] = stored_hash
        return True
```

### HIPAA Break-Glass Emergency Access

```python
class HIPAAAccessControl:
    """RBAC with MFA and break-glass emergency access."""

    FIELD_ACCESS = {
        "physician": {"*"},  # All fields for assigned patients
        "nurse": {"demographics", "vitals", "medications", "allergies"},
        "pharmacist": {"medications", "allergies", "prescriptions"},
        "billing": {"demographics", "insurance", "billing_codes"},
        "admin": set(),  # No PHI access
        "emergency": {"*"},  # Full access during break-glass (logged)
    }

    def check_access(self, user, patient_id, fields, purpose):
        if not user.mfa_verified:
            return AccessDecision(allowed=False, denial="MFA required")
        allowed = self.FIELD_ACCESS.get(user.role, set())
        unauthorized = [f for f in fields if "*" not in allowed and f not in allowed]
        if unauthorized:
            return AccessDecision(allowed=False, denial=f"Cannot access: {unauthorized}")
        if not self._has_care_relationship(user.id, patient_id):
            return AccessDecision(allowed=False, denial="No care relationship")
        if user.last_activity < datetime.utcnow() - timedelta(minutes=15):
            return AccessDecision(allowed=False, denial="Session expired (15min)")
        return AccessDecision(allowed=True)

    def activate_break_glass(self, user, reason):
        """Emergency PHI access — auto-expires 4h, reviewed within 24h."""
        access = BreakGlassAccess(
            user_id=user.id, reason=reason,
            started_at=datetime.utcnow(),
            expires_at=datetime.utcnow() + timedelta(hours=4),
        )
        self.alert_security_team(f"BREAK-GLASS: {user.name} — {reason}")
        self.scheduler.schedule_revoke(access.access_id, access.expires_at)
        return access
```

---

## GDPR Technical Implementation

### Consent Management (TCF 2.0)

```
Consent Management Platform (CMP):
  Purposes (10): Store/access device info, Basic ads, Personalised ads,
    Personalised content, Measurement, Improve products, Market research,
    Data sharing, Data linking, Precise geolocation

  Legal Bases: Consent (Art. 6(1)(a)), Legitimate Interest (Art. 6(1)(f))

  Consent string format: TCF v2
  Stored: first-party cookie or localStorage
```

```javascript
function canProcessData(purposeId, vendorId) {
  const consentString = getTCConsentString();
  const tcModel = TCString.decode(consentString);
  if (!tcModel.purposeConsents.has(purposeId)) return false;
  if (!tcModel.vendorConsents.has(vendorId)) return false;
  return true;
}
```

### Data Minimization Enforcement

```python
class DataMinimizationValidator:
    """Validate only necessary data collected per Art. 5(1)(c)."""

    REQUIRED_PURPOSES = {
        "registration": ["email", "name"],
        "payment": ["email", "billing_address", "payment_token"],
        "analytics": ["session_id", "page_views"],  # No PII
    }

    def validate_collection(self, purpose: str, fields: dict) -> list:
        findings = []
        allowed = self.REQUIRED_PURPOSES.get(purpose, [])
        for field_name in fields:
            if field_name not in allowed:
                findings.append(Finding(
                    severity="high", article="Art. 5(1)(c)",
                    description=f"Unnecessary: '{field_name}' for '{purpose}'",
                ))
        return findings
```

### DPIA (Data Protection Impact Assessment)

DPIA required when (Art. 35):
- [ ] Systematic, extensive profiling with significant effects
- [ ] Large-scale processing of special category data (Art. 9)
- [ ] New technology with high privacy risk
- [ ] Automated decision-making with legal effects
- [ ] Cross-border transfers without adequacy decision

DPIA structure: Processing description → Necessity assessment → Risk assessment → Mitigation measures → DPO consultation → Annual review

---

## BeyondCorp — Google's Zero Trust

### Architecture Components

| Component | Function |
|-----------|----------|
| Device Inventory Service | Tracks all managed devices (serial, owner, patch level, EDR) |
| Access Proxy | Single entry point, TLS termination, auth, policy enforcement |
| Access Control Engine | Central policy evaluation (identity + device trust + context) |
| Continuous Evaluation | Re-evaluate sessions on signal change |

### Trust Tiers

| Tier | Device Status | User Auth | Access Level |
|------|--------------|-----------|--------------|
| Full Trust | Managed, patched, encrypted, EDR active | MFA recent (<8h) | All apps |
| Partial Trust | Managed, some drift or off-network | MFA older (>8h) | Most apps, not crown jewels |
| Untrusted | Unmanaged or non-compliant | Basic auth | Public apps only, read-only |

**BeyondCorp checklist:**
- [ ] Device inventory covers all endpoints
- [ ] Access proxy is single entry point (no bypass)
- [ ] Trust tier computed dynamically, not cached indefinitely
- [ ] Continuous re-evaluation during active sessions
- [ ] Sensitive apps require highest trust tier
- [ ] No VPN dependency for internal app access
- [ ] Session termination on trust tier drop

---

## Dockerfile Security Examples

```dockerfile
# BAD: Running as root, no multi-stage, unpinned base
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
# GOOD: Non-root, multi-stage, pinned, minimal
FROM node:20.11-alpine3.19 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --ignore-scripts && npm cache clean --force
COPY . .
RUN npm run build

FROM node:20.11-alpine3.19
RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -s /bin/sh -D appuser
WORKDIR /app
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

---

## Kubernetes Security Context Examples

```yaml
# GOOD: Restricted security context
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    resources:
      limits:
        memory: "256Mi"
        cpu: "500m"
      requests:
        memory: "128Mi"
        cpu: "250m"
```

---

## Performance Profiling Tools

```bash
# Python — profiling
python -m cProfile -o profile.prof script.py
python -m snakeviz profile.prof  # Visualize

# Python — memory profiling
pip install memory_profiler
python -m memory_profiler script.py

# Python — concurrency analysis
py-spy record -o profile.svg -- python script.py

# Go — race detector
go test -race ./...

# Java — thread dump
jstack <pid> > thread_dump.txt
```

---

## Cloud Misconfig Detection Scripts

```bash
# Find public S3 buckets
aws s3api list-buckets --query 'Buckets[].Name' --output text | while read bucket; do
  acl=$(aws s3api get-bucket-acl --bucket "$bucket" 2>/dev/null)
  if echo "$acl" | grep -q "AllUsers"; then
    echo "CRITICAL: $bucket is publicly accessible"
  fi
done

# Find security groups with 0.0.0.0/0
aws ec2 describe-security-groups \
  --filters Name=ip-permission.cidr,Values='0.0.0.0/0' \
  --query 'SecurityGroups[*].[GroupId,GroupName]' --output table
```

---

## Review Metrics

**Process metrics:**
- Review turnaround time (request to first response)
- Total review cycle time (request to merge)
- PR size distribution (lines changed, files touched)
- Review iteration count (comments-to-merge cycles)

**Quality metrics:**
- Defect escape rate (post-merge bugs vs during review)
- Comment density (comments per 100 lines changed)
- Post-merge revert rate

**Research findings (SmartBear 2024):**
- Best defect detection: patches under 400 lines
- Diminishing returns after 200-400 LOC per review
- Review rate >500 LOC/hour drops defect detection
- Sweet spot: 60-90 min review sessions

## Related Skills

  - [sdlc-testing-qa](sdlc-testing-qa): Test pyramid (unit/integration/e2e), TDD/BDD, property-based testing, mutation testing, contract tes
  - [sdlc-legal-compliance](sdlc-legal-compliance): Software company legal and compliance: GDPR, SOC 2, CCPA, privacy policy, terms of service, data pro
  - [sdlc-deployment](sdlc-deployment): Deployment strategies: canary, blue-green, rolling, progressive delivery, feature flags, rollback, d
