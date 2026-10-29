<!-- masterno/agent-skills:improve-codebase-architecture/SKILL.md | https://github.com/masterno/agent-skills/blob/HEAD/improve-codebase-architecture/SKILL.md -->

---
name: improve-codebase-architecture
description: Explore a codebase to find opportunities for architectural improvement, focusing on making the codebase more testable by deepening shallow modules. Use when user wants to improve architecture, find refactoring opportunities, consolidate tightly-coupled modules, or make a codebase more AI-navigable.
---

# Improve Codebase Architecture

Explore a codebase like an AI would, surface architectural friction, discover opportunities for improving testability, and propose module-deepening refactors.

A **deep module** (John Ousterhout, "A Philosophy of Software Design") has a small interface hiding a large implementation. Deep modules are more testable, more AI-navigable, and let you test at the boundary instead of inside.

If this skill is active alongside a scope-control or implementation-discipline skill such as `$agent-drift-guard`, read [references/skill-interactions.md](references/skill-interactions.md) before moving from exploration into design or implementation planning.

## Process

### 1. Ground yourself

Before exploring friction, do a short grounding pass:

- Read the feature or package README/docs if present
- Identify the main runtime entry points, route boundaries, or top-level services for the area you are inspecting
- Note any obvious ownership boundaries already present in the codebase

Do not spend long here. The goal is to avoid mistaking unfamiliarity for architectural friction.

### 2. Explore the codebase

Explore the codebase naturally. Use explorer sub-agents for bounded read-only investigation when that speeds things up. In current Codex environments, prefer `spawn_agent` with `agent_type=explorer` for parallel codebase questions, but keep critical-path exploration local when faster.

Do NOT follow rigid heuristics - explore organically and note where you experience friction:

- Where does understanding one concept require bouncing between many small files?
- Where are modules so shallow that the interface is nearly as complex as the implementation?
- Where have pure functions been extracted just for testability, but the real bugs hide in how they're called?
- Where do tightly-coupled modules create integration risk in the seams between them?
- Which parts of the codebase are untested, or hard to test?

The friction you encounter IS the signal.

### 3. Present candidates

Present a numbered list of deepening opportunities. Rank them from strongest to weakest opportunity. For each candidate, show:

- **Cluster**: Which modules/concepts are involved
- **Why they're coupled**: Shared types, call patterns, co-ownership of a concept
- **Dependency category**: See [REFERENCE.md](REFERENCE.md) for the four categories
- **Test impact**: What existing tests would be replaced by boundary tests
- **Expected payoff**: What gets simpler, safer, or easier to change
- **Migration complexity**: How hard the refactor is to land
- **Regression risk**: Where behavior could accidentally change
- **Estimated scope**: Small, medium, or large

Do NOT propose interfaces yet. Ask the user: "Which of these would you like to explore?"

### 4. User picks a candidate

### 5. Frame the problem space

Before spawning sub-agents, write a user-facing explanation of the problem space for the chosen candidate:

- The constraints any new interface would need to satisfy
- The dependencies it would need to rely on
- A rough illustrative code sketch to make the constraints concrete - this is not a proposal, just a way to ground the constraints

Show this to the user, then immediately proceed to Step 6. The user reads and thinks about the problem while the sub-agents work in parallel.

### 6. Design multiple interfaces

Design multiple interfaces for the deepened module. Use 2 designs for smaller or more obvious cases, and 3-4 designs for high-impact or ambiguous cases.

If you use sub-agents, prefer parallel workers with clearly different design constraints. In current Codex environments, use `spawn_agent` and give each sub-agent a separate technical brief (file paths, coupling details, dependency category, what's being hidden). This brief is independent of the user-facing explanation in Step 5.

Give each design a different constraint:

- Design 1: "Minimize the interface - aim for 1-3 entry points max"
- Design 2: "Maximize flexibility - support many use cases and extension"
- Design 3: "Optimize for the most common caller - make the default case trivial"
- Design 4 (if applicable): "Design around the ports & adapters pattern for cross-boundary dependencies"

Each design should include:

1. Interface signature (types, methods, params)
2. Usage example showing how callers use it
3. What complexity it hides internally
4. Dependency strategy (how deps are handled - see [REFERENCE.md](REFERENCE.md))
5. Trade-offs

Present designs sequentially, then compare them in prose.

After comparing, give your own recommendation: which design you think is strongest and why. If elements from different designs would combine well, propose a hybrid. Be opinionated - the user wants a strong read, not just a menu.

### 7. User picks an interface (or accepts recommendation)

### 8. Produce an implementation brief

Produce a concise implementation brief using the template in [REFERENCE.md](REFERENCE.md). The brief should be durable enough that someone could implement the refactor later without repeating the full exploration.

Include:

- Affected files/modules
- Target module responsibilities and boundary
- Migration strategy for callers
- Tests to add, update, or delete
- Rollout risks and validation points

Share the brief with the user as the final output of the skill.
