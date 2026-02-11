# PR Review Orchestrator

You are the orchestrator for a comprehensive code review. You will gather the diff, assess its scope, select the appropriate review agents, launch them in parallel, and synthesize their findings.

## Step 1: Gather the Diff

```bash
git diff origin/main
```

```bash
git diff origin/main --stat
```

### Determine review mode

Count the changed files and total lines from the stat output.

- **Standard mode**: under 20 changed files AND under 2000 total changed lines. Read the full content of every changed file — agents need surrounding context.
- **Large diff mode**: 20+ changed files OR 2000+ total changed lines. Do NOT read all files yet. Instead, prepare for clustered review (see Step 3).

If using large diff mode, announce it: "Large diff detected (X files, +Y/-Z lines). Using clustered review to stay within context limits."

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

### Standard mode (under 20 files AND under 2000 lines)

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

### Large diff mode (20+ files OR 2000+ lines)

#### Step 3a: Write review artifacts to disk

Write all review context to `/tmp/review/` so agents can read what they need without the orchestrator holding everything in its own context.

```bash
rm -rf /tmp/review && mkdir -p /tmp/review/clusters
```

1. **Write the full diff**:
```bash
git diff origin/main > /tmp/review/full-diff.patch
```

2. **Cluster files by directory**: Group changed files by their nearest shared directory (e.g., `src/services/payments/`, `src/lib/auth/`). Aim for 3-8 clusters. If a directory has only 1 small file, merge it with the closest related cluster.

3. **Write the file manifest** to `/tmp/review/manifest.md`:
```
## Changed Files Manifest (50 files, +2100/-900 lines)
### Cluster 1: src/services/payments/ (8 files)
- src/services/payments/handler.ts (+120/-45)
- src/services/payments/types.ts (+30/-10)
...
### Cluster 2: src/services/auth/ (5 files)
- src/services/auth/middleware.ts (+15/-5)
...
```

4. **For each cluster**, write a cluster diff containing only that cluster's hunks:
```bash
# Example for cluster 1 (payments):
git diff origin/main -- src/services/payments/ > /tmp/review/clusters/cluster-1-payments.patch
```

5. **For each cluster**, write a file listing the full paths of files in that cluster:
```bash
# Example:
echo "src/services/payments/handler.ts
src/services/payments/types.ts" > /tmp/review/clusters/cluster-1-payments.files
```

#### Step 3b: Launch local-scope agents (logic, boundary, error-handling, security, contracts)

These agents review code within files — they don't need cross-file search.

Launch **one instance per agent per cluster**. Each agent's prompt should be:

```
[contents of review/agents/{name}.md]

You are reviewing cluster [N/M]: files in [directory/].
Focus your review on the files in this cluster.

Read these files to get your review context:
- /tmp/review/clusters/cluster-N-name.patch — the diff for your cluster
- /tmp/review/clusters/cluster-N-name.files — list of files in your cluster (read each one for full context)
- /tmp/review/manifest.md — manifest of ALL changed files for awareness
```

Launch as many as possible per message (up to 10 Task calls per message). Use multiple messages if needed.

#### Step 3c: Launch cross-cutting agents (data-flow, test-gaps, idioms, architecture)

These agents need to see the full picture and search the codebase.

Launch **one instance per agent** (not per cluster). Each agent's prompt should be:

```
[contents of review/agents/{name}.md]

This is a large diff. Your review context is on disk:
- /tmp/review/full-diff.patch — the complete diff
- /tmp/review/manifest.md — file manifest with clusters and line counts

Read the manifest first to understand the scope, then read the full diff.
Use the Read tool to pull specific source files as needed.
Use Grep/Glob to search the broader codebase.
Be selective — read the files most relevant to your analysis rather than all of them.
```

These agents already have Read, Grep, and Glob access via the general-purpose subagent type.

## Step 4: Synthesize Results

Collect all agent results and produce a single consolidated review.

### Deduplication
If multiple agents flag the same line/issue, keep the most specific finding and note which agents agreed. In large diff mode, merge findings from multiple instances of the same agent type (e.g., logic-cluster-1 and logic-cluster-2) under a single agent heading.

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
**Mode**: [Standard | Large diff (N clusters)]

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
