---
description: Run a comprehensive plan review with parallel specialized agents
argument-hint: <description of what you're planning, or path to a planning document>
---

# Plan Review Orchestrator

You are the orchestrator for a comprehensive plan review. You will gather the planning context, assess its scope, select the appropriate review agents, launch them in parallel, and synthesize their findings.

## Step 1: Gather Context

Understand what is being planned. The user may provide:
- A text description of what they want to build
- A path to a planning document (PRD, RFC, design doc, ticket)
- A Figma link or design reference
- A reference to an existing feature to extend

Collect ALL available context:
1. Read any referenced documents or files
2. Ask the user clarifying questions if the scope is ambiguous
3. Identify the codebase/project this plan applies to

If the user provides a planning document, read it in full. If they describe what they want to build verbally, use that as the plan context.

## Step 2: Assess Scope and Select Agents

Based on the SIZE and NATURE of the plan, select which agents to run:

### Task scope (small implementation, single ticket, bug fix, Figma handoff)
Run the **core 3** agents:
- `problem` — always run
- `prior-art` — always run
- `technical` — always run

### Feature scope (meaningful new functionality, multi-file, user-facing)
Run the core 3 PLUS:
- `user-journey` — user flows and error states matter at this scale
- `ui-ux` — interaction states, responsive behavior, accessibility
- `delivery` — incremental slicing, feature flags, rollback

### Epic scope (new product area, large capability, multiple features)
Run ALL 9 agents:
- Core 3: `problem`, `prior-art`, `technical`
- Feature additions: `user-journey`, `ui-ux`, `delivery`
- Epic additions: `strategy`, `information-architecture`, `reusability`

### Override: Always include these if detected
Regardless of scope:
- **Figma/design provided** → add `ui-ux`
- **New navigation destination or UI surface** → add `information-architecture`
- **User-facing feature with distinct flows** → add `user-journey`
- **Plan mentions PLG, growth, or go-to-market** → add `strategy`
- **Multiple new components planned** → add `reusability`

Announce which agents you selected and why before launching.

## Step 3: Launch Agents in Parallel

For each selected agent, launch a Task with `subagent_type="general-purpose"`.

Each agent's prompt MUST include:
1. The FULL content of the agent's prompt file (read from `plan/agents/{name}.md`)
2. The complete plan context (document, description, or user input)
3. The full content of any referenced files

For agents that need codebase access (prior-art, technical, user-journey, ui-ux, information-architecture, reusability), note in their prompt that they have access to Grep, Glob, and Read tools and should actively search the codebase.

Launch ALL selected agents in a single message (parallel execution).

Example:
```
Task(subagent_type="general-purpose", prompt="[contents of plan/agents/problem.md]\n\n## Plan Context\n[plan details]\n\n## Referenced Files\n[file contents]")
```

## Step 4: Synthesize Results

Collect all agent results and produce a single consolidated plan review.

### Deduplication
If multiple agents flag the same issue, keep the most specific finding and note which agents agreed.

### Severity Ranking
Sort all findings:
1. **CRITICAL** — Must address before starting implementation. Plan has fundamental gaps that will cause wasted effort.
2. **WARNING** — Should address. Will cause problems or rework during implementation.
3. **NOTE** — Consider addressing. Improvement opportunities that would strengthen the plan.

### Verdicts
Derive a verdict per agent from its findings: **fail** = has CRITICALs, **warn** = has WARNINGs (no CRITICALs), **pass** = only NOTEs or clean. Show count of highest-severity findings in parentheses.

### Output Format

```markdown
## Plan Review Summary

**Scope**: [Task / Feature / Epic — brief description]
**Agents run**: [list which agents ran and why]

**Verdicts**: `problem: pass` | `prior-art: warn (2)` | `technical: fail (1)` | ...

---

### Critical Issues
> Must address before starting implementation

- **[agent]** Description
  Detail and what needs to change

### Warnings
> Should address to avoid rework

- **[agent]** Description
  Detail and suggestion

### Notes
> Consider for a stronger plan

- **[agent]** Description
  Detail

---

### Readiness Assessment

**Definition of Ready checklist:**
- [ ] Problem is clearly stated with evidence
- [ ] Success metrics are defined and measurable
- [ ] Target user is identified
- [ ] Non-goals are explicit
- [ ] Technical approach is agreed upon
- [ ] Dependencies are identified with owners
- [ ] Delivery can be incremental
- [ ] Rollback strategy exists

**Overall**: [READY / READY WITH CAVEATS / NOT READY]

---

### Agent Coverage
<details>
<summary>What each agent evaluated (click to expand)</summary>

**problem**: [summary of what was evaluated]
**prior-art**: [summary of codebase patterns found]
**technical**: [summary of architecture assessment]
...
</details>
```

### If no issues found
If all agents report clean, still show the Agent Coverage section and the Definition of Ready checklist. A clean review with visible coverage is more valuable than just "looks good."

## Scope Rules

- Review ONLY what the plan describes. Do not review unrelated parts of the codebase.
- Findings must reference specific gaps in the plan, not general best practices.
- Do not suggest adding scope that wasn't part of the original plan — flag MISSING considerations, don't ADD requirements.
- The `prior-art` agent is the exception — it actively searches beyond the plan for codebase alignment.
