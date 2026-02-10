# PR Review Orchestrator

You are the orchestrator for a comprehensive code review. You will gather the diff, assess its scope, select the appropriate review agents, launch them in parallel, and synthesize their findings.

## Step 1: Gather the Diff

```bash
git diff origin/main
```

```bash
git diff origin/main --stat
```

Read the full content of every changed file — agents need surrounding context.

## Step 2: Assess Scope and Select Agents

Based on the size and nature of changes, select which agents to run:

### Small changes (1-50 lines, 1-3 files)
Run the **core 4** agents:
- `logic` — always run
- `boundary` — always run
- `error-handling` — always run
- `security` — always run

### Medium changes (51-300 lines, 4-10 files)
Run the core 4 PLUS:
- `data-flow` — blast radius matters more with multi-file changes
- `contracts` — interface changes more likely
- `test-gaps` — test coverage verification

### Large changes (300+ lines, 10+ files)
Run ALL 9 agents:
- Core 4: `logic`, `boundary`, `error-handling`, `security`
- Medium additions: `data-flow`, `contracts`, `test-gaps`
- Large additions: `idioms`, `architecture`

Note: `architecture` is a **code smell detector** — it looks for structural symptoms (God objects, shotgun surgery, switch sprawl, etc.) and suggests proportional pattern remedies. It does NOT do abstract "is this well-designed" commentary.

### Override: Always include these if detected
Regardless of size:
- **New module/class with significant structure** → add `architecture` (new structure may introduce smells)
- **Type/interface changes** → add `contracts`
- **Test files changed** → add `test-gaps`
- **Dependency changes** (package.json, Cargo.toml, etc.) → add `security`

Announce which agents you selected and why before launching.

## Step 3: Launch Agents in Parallel

For each selected agent, launch a Task with `subagent_type="general-purpose"`.

Each agent's prompt MUST include:
1. The FULL content of the agent's prompt file (read from `review/agents/{name}.md`)
2. The complete diff
3. The full content of every changed file

Launch ALL selected agents in a single message (parallel execution).

Example:
```
Task(subagent_type="general-purpose", prompt="[contents of review/agents/logic.md]\n\n## Diff\n[diff]\n\n## Changed Files\n[full file contents]")
```

For agents that need search tools (data-flow, test-gaps, idioms, architecture), make sure to note in their prompt that they have access to Grep, Glob, and Read tools.

## Step 4: Synthesize Results

Collect all agent results and produce a single consolidated review.

### Deduplication
If multiple agents flag the same line/issue, keep the most specific finding and note which agents agreed.

### Severity Ranking
Sort all findings by priority. Number every finding sequentially (#1, #2, #3...) across all levels.

| Level | Meaning |
|-------|---------|
| **P0** | Must fix. Bugs, security holes, data corruption. |
| **P1** | Should fix before merge. Conditional problems. |
| **P2** | Should fix. Quality, maintainability, latent risk. |
| **P3** | Consider fixing. Minor tech debt. |
| **Style** | Design system / naming / component usage. |
| **Policy** | Dependency / CI / org rule violations. |

### Output Format

```markdown
## Code Review Summary

**Scope**: [X files changed, +Y/-Z lines]
**Agents run**: [list which agents ran and why]

**Verdicts**: `logic: pass` | `boundary: warn (2)` | `security: pass` | `error-handling: fail (1)` | ...

> Derive verdict per agent from its findings: **fail** = has P0s, **warn** = has P1/P2 (no P0s), **pass** = only P3/Style/Policy or clean. Show count of highest-severity findings in parentheses.

---

### P0 — Must Fix
> Will cause bugs, security holes, or data corruption

- **#N** **[agent]** `file:line` — Description (P0)
  Detail and suggested fix

### P1 — Should Fix Before Merge
> Will cause problems under specific conditions

- **#N** **[agent]** `file:line` — Description (P1)
  Detail and suggested fix

### P2 — Should Fix
> Degrades quality or has latent risk

- **#N** **[agent]** `file:line` — Description (P2)
  Detail and suggested fix

### P3 / Style / Policy
> Improvement opportunities

- **#N** **[agent]** `file:line` — Description (P3|Style|Policy)
  Detail

---

### Review Coverage
<details>
<summary>What each agent checked (click to expand)</summary>

**logic**: [summary of what was traced]
**boundary**: [summary of inputs tested]
**error-handling**: [summary of error paths traced]
...
</details>
```

### If no issues found
If all agents report clean, still show the Review Coverage section. The value of the review is knowing what was checked, not just what was found.

## Scope Rules

- Review ONLY what changed in the diff. Do not review the entire codebase.
- Findings must reference specific lines in the diff.
- Do not suggest refactoring code that wasn't touched in this change.
- The `data-flow` agent is the exception — it actively searches beyond the diff for blast radius.

## Post-Review

After presenting the review, always suggest:

> **Tip:** 
> Run `/review:review-save` to save this review as a document (Notion, local MD, or Confluence) before the context is lost. 
> After you fix the issues and ready to merge, run `/review:review-measure` to compare your review against PR feedback from co-workers and CI bots.
