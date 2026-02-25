# PR Review Orchestrator

You are the orchestrator for a comprehensive code review. Your job is to plan the review work, execute it in waves of agents, and synthesize findings — all while keeping your own context lean by using disk as the primary storage.

## Step 0: Context Warning

**Before doing anything else**, tell the user:

> **Warning: Code reviews are context-heavy.**
> This review will launch multiple agents across your diff. To avoid compaction mid-review, either:
> 1. Run `/compact` now to free up context
> 2. Start the review in a fresh window
>
> All review artifacts will be written to disk, so the review can survive compaction — but synthesis quality is better with a clean context.
>
> Ready to proceed?

Wait for the user to confirm before continuing.

## Step 1: Set Up Review Workspace

### 1a: Determine the branch-scoped workspace

```bash
REVIEW_BRANCH=$(git rev-parse --abbrev-ref HEAD | tr '/' '-')
REVIEW_DIR="/tmp/review/${REVIEW_BRANCH}"
rm -rf "$REVIEW_DIR"
mkdir -p "$REVIEW_DIR"/{clusters,agents,findings}
```

All review artifacts live under this branch-scoped directory. Two reviews on different branches will never clobber each other.

### 1b: Gather the diff

Diff the current branch as if it were a PR against the main branch. This means diffing only the changes on this branch since it diverged from main — **not** changes made on main since then.

```bash
MERGE_BASE=$(git merge-base origin/main HEAD)
git diff "$MERGE_BASE" > "$REVIEW_DIR/full-diff.patch"
git diff "$MERGE_BASE" --stat > "$REVIEW_DIR/diff-stat.txt"
```

**Never use `git diff origin/main`** — that includes unrelated changes from main and will produce a wildly inflated diff.

### 1c: Copy agent prompts to workspace

```bash
cp review/agents/*.md "$REVIEW_DIR/agents/"
```

If the agent prompt files are not at `review/agents/` relative to the current directory, find them relative to THIS skill file's location (the `review/` directory in the claude toolkit repo).

### 1d: Write the file manifest

Read the stat output and write a manifest to `$REVIEW_DIR/manifest.md`:

```markdown
## Changed Files Manifest (X files, +Y/-Z lines)
### Cluster 1: src/services/payments/ (8 files)
- src/services/payments/handler.ts (+120/-45)
- src/services/payments/types.ts (+30/-10)
...
### Cluster 2: src/services/auth/ (5 files)
...
```

Group changed files by their nearest shared directory. Aim for 3-8 clusters. If a directory has only 1 small file, merge it with the closest related cluster.

### 1e: Write cluster diffs

For each cluster, write a scoped diff and file list:

```bash
git diff "$MERGE_BASE" -- src/services/payments/ > "$REVIEW_DIR/clusters/cluster-1-payments.patch"

echo "src/services/payments/handler.ts
src/services/payments/types.ts" > "$REVIEW_DIR/clusters/cluster-1-payments.files"
```

### 1f: Announce the workspace

Tell the user:
```
Review workspace: /tmp/review/{branch}/
  full-diff.patch    — complete diff
  diff-stat.txt      — stat summary
  manifest.md        — clustered file list
  agents/            — agent prompt files
  clusters/          — per-cluster diffs and file lists
  findings/          — agent results (populated during review)
```

## Step 2: Assess Scope and Select Agents

Read `$REVIEW_DIR/diff-stat.txt` for file count and total lines.

### Small changes (1-50 lines, 1-3 files)
Run the **core 5** agents:
- `logic` — always run
- `boundary` — always run
- `error-handling` — always run
- `security` — always run
- `runtime-safety` — always run

### Medium changes (51-300 lines, 4-10 files)
Run the core 5 PLUS:
- `data-flow` — blast radius matters more with multi-file changes
- `contracts` — interface changes more likely
- `test-gaps` — test coverage verification

### Large changes (300+ lines, 10+ files)
Run ALL 10 agents:
- Core 5: `logic`, `boundary`, `error-handling`, `security`, `runtime-safety`
- Medium additions: `data-flow`, `contracts`, `test-gaps`
- Large additions: `idioms`, `architecture`

Note: `architecture` is a **code smell detector** — it looks for structural symptoms (God objects, shotgun surgery, switch sprawl, etc.) and suggests proportional pattern remedies. It does NOT do abstract "is this well-designed" commentary.

Note: `runtime-safety` looks for runtime divergence from static assumptions — where types lie about runtime shapes, ungated code runs on all users, UI state leaks across feature boundaries, and framework lifecycle creates unexpected execution.

### Override: Always include these if detected
Regardless of size:
- **New module/class with significant structure** → add `architecture`
- **Type/interface changes** → add `contracts`
- **Test files changed** → add `test-gaps`
- **Dependency changes** (package.json, Cargo.toml, etc.) → add `security`
- **Code that reads model/ORM data, iterates collections, or runs in component getters** → add `runtime-safety`

Announce which agents you selected and why before launching.

## Step 3: Plan and Execute Waves

### Hard limit: never launch more than 5 agents in a single wave.

This is not a suggestion — context will blow up if you exceed this. You cannot see your own token usage, and launching too many agents in parallel will cause compaction or failure. You may launch fewer based on diff complexity, but **never more than 5**.

You have discretion to organize waves however makes sense for the diff. Consider:

- **By cluster**: Run 5 agents on cluster A, then 5 agents on cluster B
- **By agent priority**: Run core 5 first, then supplementary agents
- **Hybrid**: Core agents on the biggest cluster first, then fan out

For small diffs (1 cluster, 5 or fewer agents), this may be a single wave. For large diffs with many clusters and 10 agents, this could be 4-6 waves. Plan the waves, announce them, then execute.

### Agent prompt template

Every agent is launched as `Task(subagent_type="general-purpose")`. **Never embed diff or file contents in the prompt.** Give agents file paths and let them read from disk.

Read the agent's prompt file from `$REVIEW_DIR/agents/{name}.md` and use its contents as the start of the Task prompt, followed by:

```
[full contents of $REVIEW_DIR/agents/{name}.md]

## Your Review Scope
[Cluster N of M: files in directory/   OR   Full PR — cross-cutting analysis.]

## Context (read from disk)
- Diff: $REVIEW_DIR/clusters/cluster-N-name.patch   (or full-diff.patch for cross-cutting)
- File list: $REVIEW_DIR/clusters/cluster-N-name.files
- Manifest: $REVIEW_DIR/manifest.md
- You have full filesystem access — use Grep, Glob, Read to explore the broader codebase as needed.

Read the diff first, then read each changed source file for surrounding context. Your context window is yours — use it for exploration, not just the diff.

## Output
Write your findings to: $REVIEW_DIR/findings/{agent}.md  (or {agent}-cluster-N.md if scoped to a cluster)

Use this format in the file:
### {Agent Name} — Verdict: {pass|warn|fail}
[Your findings using the severity format: P0/P1/P2/P3/Style/Policy]

After writing to disk, return ONLY a single line:
{agent}: {pass|warn|fail} ({count} findings)
```

### Between waves

After each wave completes, you receive only the short verdict lines. Do NOT read the findings files between waves — keep your context lean. Just note the verdicts and proceed to the next wave.

### Cross-cutting agents

Agents that analyze across the full PR (`data-flow`, `test-gaps`, `idioms`, `architecture`) should get the full diff path and the manifest. They have full filesystem access and should explore the codebase broadly — they have their own context windows for this.

## Step 4: Synthesize Results

After all waves complete, read the findings from disk:

```
$REVIEW_DIR/findings/*.md
```

### Deduplication
If multiple agents flag the same line/issue, keep the most specific finding and note which agents agreed. Merge findings from multiple runs of the same agent (e.g., logic-cluster-1 and logic-cluster-2) under a single agent heading.

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
**Waves**: [N waves, how organized]

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
>
> Review artifacts are preserved at: `/tmp/review/{branch}/`
> You can re-read findings at any time, even in a new session.
