<!-- gabrielrdga/n8n-skills:skills/n8n-builder-orchestrator/SKILL.md | https://github.com/gabrielrdga/n8n-skills/blob/HEAD/skills/n8n-builder-orchestrator/SKILL.md -->

---
name: n8n-builder-orchestrator
description: End-to-end workflow builder orchestration. Use when building complete workflows from scratch, orchestrating the full build pipeline, or managing the discovery-to-deploy lifecycle. This is the meta-skill that coordinates all other skills.
---

# n8n Builder Orchestrator

Meta-skill that orchestrates the complete workflow build lifecycle.

---

## Orchestrator Role

The orchestrator is the **conductor** — it doesn't build workflows directly but coordinates all other skills in the right sequence. Think of it as the project manager for workflow construction.

**Input**: A user request or a Workflow Spec (from PRD Discovery)
**Output**: A fully built, tested, audited, and published workflow

---

## The Build Pipeline

```
STAGE 1: DISCOVERY
   │ n8n-prd-discovery
   │ → Workflow Spec YAML
   ↓
STAGE 2: CREDENTIAL SETUP
   │ n8n-credential-policy
   │ → Credentials verified/created
   ↓
STAGE 3: SCAFFOLD
   │ n8n-workflow-patterns
   │ → Workflow created with correct pattern
   ↓
STAGE 4: CONFIGURE
   │ n8n-node-configuration + n8n-mcp-tools-expert
   │ → Nodes configured with correct params
   ↓
STAGE 5: CODE (if needed)
   │ n8n-code-javascript / n8n-code-python
   │ → Code nodes with correct patterns
   ↓
STAGE 6: EXPRESSIONS
   │ n8n-expression-syntax
   │ → Data mappings correct
   ↓
STAGE 7: VALIDATE
   │ n8n-validation-expert
   │ → All validation errors resolved
   ↓
STAGE 8: TEST GATE
   │ n8n-test-gate
   │ → 6-gate quality check passed
   ↓
STAGE 9: AUDIT
   │ n8n-audit-compliance
   │ → No critical/high findings
   ↓
STAGE 10: PUBLISH
   │ n8n-publish-gate
   │ → Approved and activated
   ↓
COMPLETE ✅
```

---

## Stage Details

### Stage 1: Discovery

**Skill**: n8n-prd-discovery

**Actions**:
1. If user provides a clear request → Generate Workflow Spec
2. If request is vague → Ask discovery questions
3. Search for similar templates
4. Assess complexity

**Output**: Workflow Spec YAML

**Skip if**: User provides a complete spec or is modifying an existing workflow.

---

### Stage 2: Credential Setup

**Skill**: n8n-credential-policy

**Actions**:
1. List existing credentials
2. Match against spec requirements
3. For missing credentials: getSchema → create (with policy)
4. For existing: verify they're active

**Output**: All credentials ready

**Skip if**: No external services needed.

```javascript
// Orchestrator logic
for (const cred of spec.credentials) {
  const existing = credentials.find(c => c.type === cred.type);
  if (existing) {
    cred.status = "existing";
    cred.id = existing.id;
  } else {
    // Follow credential-policy for creation
    const schema = await n8n_manage_credentials({
      action: "getSchema",
      credentialType: cred.type
    });
    // Present required fields to user
    console.log(`Need credential for ${cred.type}:`);
    console.log(`Required fields: ${JSON.stringify(schema)}`);
  }
}
```

---

### Stage 3: Scaffold

**Skill**: n8n-workflow-patterns

**Actions**:
1. Select pattern based on spec
2. Search for matching template (optional)
3. Create workflow with trigger + basic structure

```javascript
// Create base workflow
const workflow = await n8n_create_workflow({
  name: spec.name,
  nodes: [triggerNode],
  connections: {}
});

// Or deploy from template
const deployed = await n8n_deploy_template({
  templateId: matchingTemplate.id,
  name: spec.name,
  autoFix: true
});
```

**Output**: Workflow ID with basic structure

---

### Stage 4: Configure

**Skills**: n8n-node-configuration + n8n-mcp-tools-expert

**Actions**:
1. For each integration in spec: `search_nodes` → `get_node`
2. Add nodes to workflow: `n8n_update_partial_workflow`
3. Configure each node based on operation requirements
4. Connect nodes

**Pattern**: Iterative — add node, validate, fix, continue.

```javascript
// Iterative building (avg 56s between edits)
for (const integration of spec.integrations) {
  // Find node
  const nodes = await search_nodes({ query: integration.service });
  const nodeInfo = await get_node({ nodeType: nodes[0].nodeType });
  
  // Add to workflow
  await n8n_update_partial_workflow({
    id: workflowId,
    intent: `Add ${integration.service} node`,
    operations: [{
      type: "addNode",
      node: { /* configured based on nodeInfo */ }
    }]
  });
  
  // Validate
  await n8n_validate_workflow({ id: workflowId });
}
```

---

### Stage 5: Code (Conditional)

**Skills**: n8n-code-javascript / n8n-code-python

**Actions** (only if spec requires custom logic):
1. Determine language (JavaScript preferred for 95% of cases)
2. Write code following established patterns
3. Handle webhook data correctly (`.body` nesting!)
4. Ensure correct return format (`[{json: {...}}]`)

---

### Stage 6: Expressions

**Skill**: n8n-expression-syntax

**Actions**:
1. Set up data mappings between nodes
2. Ensure correct `{{}}` syntax
3. Handle webhook `.body` nesting
4. Verify node references are correct

---

### Stage 7: Validate

**Skill**: n8n-validation-expert

**Actions**:
1. Run `n8n_validate_workflow` with appropriate profile
2. If errors: fix iteratively (avg 2-3 cycles)
3. If auto-fixable: use `n8n_autofix_workflow`
4. Verify clean validation

```javascript
let valid = false;
let iterations = 0;

while (!valid && iterations < 5) {
  const result = await n8n_validate_workflow({ id: workflowId });
  
  if (result.valid) {
    valid = true;
  } else {
    // Attempt auto-fix
    await n8n_autofix_workflow({
      id: workflowId,
      applyFixes: true,
      confidenceThreshold: "high"
    });
    iterations++;
  }
}
```

---

### Stage 8: Test Gate

**Skill**: n8n-test-gate

**Actions**:
1. Run structural validation
2. Check error handling
3. Run security scan
4. Execute functional test
5. Check performance considerations
6. Verify documentation

**If fails**: Return to appropriate earlier stage.

---

### Stage 9: Audit

**Skill**: n8n-audit-compliance

**Actions**:
1. Quick audit (hardcoded_secrets, unauthenticated_webhooks)
2. If findings: remediate following playbook
3. Re-audit to confirm clean

---

### Stage 10: Publish

**Skill**: n8n-publish-gate

**Actions**:
1. Verify all prerequisites
2. Check environment (auto-approve for dev, approval for prod)
3. Create version snapshot
4. Activate workflow
5. Monitor first executions

---

## Progress Reporting

After each stage, report progress:

```
BUILD PROGRESS
══════════════════════════════════
Workflow: "Order Processing Pipeline"
Started: 2026-04-11T14:00:00Z

[✅] Stage 1: Discovery         (Workflow Spec generated)
[✅] Stage 2: Credentials       (3/3 ready)
[✅] Stage 3: Scaffold          (Workflow ID: abc-123)
[✅] Stage 4: Configure         (8 nodes configured)
[✅] Stage 5: Code              (1 Code node added)
[✅] Stage 6: Expressions       (Data mappings set)
[✅] Stage 7: Validate          (Clean after 2 iterations)
[✅] Stage 8: Test Gate         (5/6 gates passed)
[🔄] Stage 9: Audit            (Running...)
[ ] Stage 10: Publish          (Pending)

Elapsed: 4m 32s
```

---

## Handling Failures

### Stage Failure → Recovery

| Failed Stage | Recovery Action |
|-------------|-----------------|
| Discovery | Ask more questions, refine spec |
| Credentials | Present required fields to user |
| Scaffold | Try different pattern or template |
| Configure | Use get_node for details, fix config |
| Code | Check error patterns, fix code |
| Expressions | Fix data paths, check .body nesting |
| Validate | Auto-fix or manual fix, re-validate |
| Test Gate | Fix failing gate, re-run |
| Audit | Remediate findings, re-audit |
| Publish | Fix blockers, re-request publish |

### Maximum Retries

- Validation: 5 iterations (then escalate)
- Auto-fix: 3 attempts (then manual)
- Test gate: 3 attempts (then review spec)
- Overall: No stage should take > 10 minutes

---

## Dry Run Mode

Preview the entire build without executing:

```
DRY RUN: "Order Processing Pipeline"
══════════════════════════════════════
Would execute 10 stages:

1. Discovery → Generate spec (webhook + Shopify + Postgres + Slack)
2. Credentials → Check 3 types (shopifyApi, postgresApi, slackApi)
3. Scaffold → Create webhook_processing pattern
4. Configure → Add 8 nodes with connections
5. Code → Add order parsing Code node
6. Expressions → Set 12 data mappings
7. Validate → Run validation with "strict" profile
8. Test Gate → Run 6 quality gates
9. Audit → Quick security scan
10. Publish → Activate with production approval

Estimated time: 5-8 minutes
Prerequisites: API credentials for all 3 services
```

---

## Best Practices

### ✅ Do
- Follow the stages in order
- Report progress after each stage
- Handle failures gracefully with recovery
- Use dry run for complex workflows
- Keep the user informed at each step

### ❌ Don't
- Skip stages (especially validation and test gate)
- Rush through without validating
- Build without discovery for complex workflows
- Ignore failing gates
- Proceed without credentials ready

---

## Related Skills

All skills in the ecosystem:
- **Layer 0**: n8n-prd-discovery
- **Layer 2**: n8n-expression-syntax, n8n-mcp-tools-expert, n8n-workflow-patterns, n8n-validation-expert, n8n-node-configuration, n8n-code-javascript, n8n-code-python
- **Layer 3**: n8n-credential-policy, n8n-test-gate, n8n-audit-compliance, n8n-publish-gate
- **Layer 4**: n8n-observability

---

## Summary

**The orchestrator knows WHAT to do and WHEN — individual skills know HOW.**

1. **Discovery** → What are we building?
2. **Credentials** → Are the keys ready?
3. **Scaffold** → Create the structure
4. **Configure** → Set up each node
5. **Code** → Custom logic (if needed)
6. **Expressions** → Wire data between nodes
7. **Validate** → Is it correct?
8. **Test** → Is it production-ready?
9. **Audit** → Is it secure?
10. **Publish** → Activate safely
