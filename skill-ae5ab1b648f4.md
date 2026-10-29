<!-- hpsgd/turtlestack:plugins/leadership/grc-lead/skills/bootstrap/SKILL.md | https://github.com/hpsgd/turtlestack/blob/HEAD/plugins/leadership/grc-lead/skills/bootstrap/SKILL.md -->

---
name: bootstrap
bootstrap-phase: governance
description: "Bootstrap the governance, risk, and compliance documentation structure for a project. Creates docs/governance/, generates initial templates, and writes the grc-lead fragment of the governance domain doc (the coordinator assembles docs/governance/CLAUDE.md). Idempotent — merges missing sections into existing files without overwriting."
argument-hint: "[project name]"
user-invocable: false
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Bootstrap Governance Documentation

Bootstrap the governance, risk, and compliance documentation structure for **$ARGUMENTS**.

## Process

### Step 1: Check and create domain directory

```bash
mkdir -p docs/governance docs/governance/_sections
```

### Step 2: Create or merge files

For each file below, apply the safe merge pattern:
- If file does not exist -> create from template
- If file exists -> read both, find sections in template missing from file, append missing sections with `<!-- Merged from grc-lead bootstrap v0.1.0 -->`

#### Fragment: `docs/governance/_sections/grc-lead.md`

`docs/governance/CLAUDE.md` is **assembled by the coordinator** from the fragments in `_sections/` — no plugin
writes it directly. Write the GRC lead's contribution as this fragment. It starts at H2 (the coordinator
generates the `# Governance Domain` H1 and a one-line intro). Create it with this content:

```markdown
## What This Domain Covers

- **Risk management** — ISO 31000 risk assessment and treatment
- **Compliance frameworks** — SOC 2, ISO 27001, GDPR, Essential Eight, NIST CSF
- **AI governance** — NIST AI RMF, EU AI Act compliance
- **Privacy** — Data Protection Impact Assessments (DPIAs)
- **Audit readiness** — evidence collection and control mapping

## Risk Management (ISO 31000)

### Risk assessment process

1. **Establish context** — scope, stakeholders, risk criteria
2. **Identify risks** — what can happen, how, and why
3. **Analyse risks** — likelihood and consequence assessment
4. **Evaluate risks** — compare against risk criteria, prioritise
5. **Treat risks** — avoid, mitigate, transfer, or accept

### Risk rating matrix

| Likelihood / Consequence | Insignificant | Minor | Moderate | Major | Catastrophic |
|--------------------------|---------------|-------|----------|-------|-------------|
| Almost certain | Medium | High | High | Extreme | Extreme |
| Likely | Medium | Medium | High | High | Extreme |
| Possible | Low | Medium | Medium | High | High |
| Unlikely | Low | Low | Medium | Medium | High |
| Rare | Low | Low | Low | Medium | Medium |

### Risk treatment options

| Option | When to Use |
|--------|------------|
| Avoid | Eliminate the activity that creates the risk |
| Mitigate | Reduce likelihood or consequence with controls |
| Transfer | Share risk via insurance, contracts, or outsourcing |
| Accept | Risk is within appetite — monitor and review |

## Compliance Frameworks

### SOC 2 Type II

Trust service criteria: Security, Availability, Processing Integrity, Confidentiality, Privacy. Maintain evidence of controls operating effectively over the audit period.

### ISO 27001

Information security management system (ISMS). Requires: risk assessment, Statement of Applicability, continuous improvement cycle (Plan-Do-Check-Act).

### GDPR

For EU personal data: lawful basis, data minimisation, storage limitation, data subject rights, breach notification (72 hours), DPIAs for high-risk processing.

### Essential Eight (ACSC)

Australian Cyber Security Centre maturity model. Target Maturity Level 2 minimum:
1. Application control 2. Patch applications 3. Configure MS Office macros 4. User application hardening 5. Restrict admin privileges 6. Patch operating systems 7. Multi-factor authentication 8. Regular backups

### NIST Cybersecurity Framework (CSF) 2.0

Six functions: Govern, Identify, Protect, Detect, Respond, Recover. Map controls to CSF categories for cross-framework alignment.

## AI Governance

### NIST AI Risk Management Framework (AI RMF)

Four functions: Govern, Map, Measure, Manage. Apply to all AI/ML systems:
- **Govern** — policies, roles, accountability for AI systems
- **Map** — context, intended use, risk identification
- **Measure** — evaluation metrics, bias testing, performance monitoring
- **Manage** — risk treatment, incident response, decommissioning

### EU AI Act

Classify AI systems by risk level:

| Risk Level | Examples | Requirements |
|------------|----------|-------------|
| Unacceptable | Social scoring, real-time biometric ID | Prohibited |
| High | HR screening, credit scoring, safety systems | Conformity assessment, human oversight |
| Limited | Chatbots, deepfakes | Transparency obligations |
| Minimal | Spam filters, AI-assisted games | No specific requirements |

## DPIA Process

Conduct a Data Protection Impact Assessment when processing is likely to result in high risk to individuals:

1. **Describe processing** — purpose, scope, data flows
2. **Assess necessity** — proportionality, lawful basis
3. **Identify risks** — to individuals' rights and freedoms
4. **Mitigations** — measures to address risks
5. **Sign-off** — DPO review and approval

## Audit Readiness

- Maintain a control register mapping controls to frameworks
- Evidence collection: automated where possible (CI logs, access reviews)
- Review cadence: quarterly internal review, annual external audit
- Gap analysis: track remediation items with owners and deadlines

## Tooling

| Tool | Purpose |
|------|---------|
| [MS 365 SharePoint](https://www.microsoft.com/en-au/microsoft-365) | Policy storage and stakeholder access |
| [GitHub](https://github.com) | Governance docs versioned in repo |

## Available Skills

| Skill | Purpose |
|-------|---------|
| `/grc-lead:risk-assessment` | Conduct an ISO 31000 risk assessment |
| `/grc-lead:compliance-audit` | Audit compliance against a framework |
| `/grc-lead:ai-governance-review` | Review AI system governance |
| `/grc-lead:write-dpia` | Write a Data Protection Impact Assessment |

## Conventions

- Every identified risk is recorded in the risk register with an owner
- Compliance evidence is collected continuously, not just before audits
- AI systems are classified per EU AI Act risk levels before deployment
- DPIAs are completed before any new high-risk personal data processing
- Policies are reviewed annually and after significant changes
- All governance documents are version-controlled in this repository
```

#### File 2: `docs/governance/risk-register.md`

Create with this content:

```markdown
# Risk Register

> Track all identified risks, their ratings, treatments, and owners.

## Active Risks

| ID | Risk Description | Category | Likelihood | Consequence | Rating | Treatment | Owner | Review Date |
|----|-----------------|----------|-----------|-------------|--------|-----------|-------|-------------|
| R-001 | | | | | | | | |

## Risk Categories

| Category | Examples |
|----------|---------|
| Strategic | Market changes, competitive threats |
| Operational | Process failures, resource constraints |
| Technical | System failures, security vulnerabilities |
| Compliance | Regulatory changes, audit findings |
| Financial | Budget overruns, revenue impact |

## Closed / Accepted Risks

| ID | Risk Description | Rating | Disposition | Closed Date |
|----|-----------------|--------|-------------|-------------|
| | | | Accepted / Mitigated / Avoided | |

> Review all active risks quarterly. Update ratings and treatments as context changes.
```

#### File 3: `docs/governance/compliance-checklist.md`

Create with this content:

```markdown
# Compliance Checklist

> Track compliance status against applicable frameworks. Update during quarterly reviews.

## Framework Applicability

| Framework | Applicable | Maturity Target | Current Status |
|-----------|-----------|-----------------|----------------|
| SOC 2 Type II | Yes / No | | Not started / In progress / Compliant |
| ISO 27001 | Yes / No | | Not started / In progress / Certified |
| GDPR | Yes / No | | Not started / In progress / Compliant |
| Essential Eight | Yes / No | ML2 / ML3 | Not started / In progress / Assessed |
| NIST CSF 2.0 | Yes / No | | Not started / In progress / Aligned |

## Control Mapping

| Control ID | Description | SOC 2 | ISO 27001 | GDPR | Essential Eight | Evidence |
|-----------|-------------|-------|-----------|------|-----------------|----------|
| | | | | | | |

## Open Gaps

| Gap ID | Framework | Control | Gap Description | Remediation | Owner | Due Date |
|--------|-----------|---------|----------------|-------------|-------|----------|
| | | | | | | |

> Map controls across frameworks to reduce duplicate effort. One control can satisfy multiple frameworks.
```

#### File 4: `docs/governance/ai-governance-policy.md`

Create with this content:

```markdown
# AI Governance Policy

> Governs the development, deployment, and operation of AI/ML systems.

## Scope

This policy applies to all AI/ML systems developed or deployed by the project, including:
- Large language model (LLM) integrations
- Machine learning models (classification, prediction, recommendation)
- Automated decision-making systems

## AI System Inventory

| System | Risk Level | Purpose | Model | Owner | Last Review |
|--------|-----------|---------|-------|-------|-------------|
| | Minimal / Limited / High | | | | |

## Principles

1. **Transparency** — users know when they are interacting with AI
2. **Fairness** — systems are tested for bias before deployment
3. **Accountability** — every AI system has a named owner
4. **Safety** — guardrails prevent harmful outputs
5. **Privacy** — personal data is minimised and protected

## Review Requirements

| Risk Level | Review Type | Frequency |
|------------|------------|-----------|
| High | Full AI governance review + DPIA | Before deployment, then annually |
| Limited | Transparency assessment | Before deployment, then annually |
| Minimal | Self-assessment | Annually |

## Incident Response

AI-specific incidents (harmful output, bias detected, data leak via AI):
1. Disable or restrict the AI system immediately
2. Notify the AI system owner and GRC lead
3. Investigate root cause and impact
4. Remediate and document in incident report
5. Update guardrails and eval suites

> Review this policy annually and after any AI-related incident.
```

### Step 3: Return manifest

After creating/merging all files, output a summary:

```
## GRC Lead Bootstrap Complete

### Files created
- `docs/governance/_sections/grc-lead.md` — grc-lead fragment (coordinator assembles `docs/governance/CLAUDE.md` from it)
- `docs/governance/risk-register.md` — risk register template
- `docs/governance/compliance-checklist.md` — compliance checklist template
- `docs/governance/ai-governance-policy.md` — AI governance policy template

### Files merged
- (list any existing files where sections were appended)

### Next steps
- Conduct initial risk assessment using `/grc-lead:risk-assessment`
- Determine applicable compliance frameworks in `compliance-checklist.md`
- Classify AI systems using `/grc-lead:ai-governance-review`
- Complete DPIAs for high-risk data processing using `/grc-lead:write-dpia`
```
