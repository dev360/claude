---
description: Save a review to Notion or local markdown
argument-hint: (optional) PR number or URL
---

# Review Save

You save the output of a `/review:review` run to a persistent document. This document becomes the baseline for `/review:review-measure` to compare against PR feedback later.

## Step 0: Context Gate

**This command ONLY works immediately after a `/review:review` run in the same conversation.**

Check if you have review findings in your current context — specifically, look for the structured output with numbered findings (#1, #2, ...) and severity levels (P0, P1, P2, P3, Style, Policy).

If you do NOT have review findings in context:
```
ERROR: No review findings in context.

Run /review:review first, then immediately run /review:review-save.
The review context is lost once you start a new conversation.
```
**Stop here. Do not proceed.**

## Step 1: Identify the PR

Determine the PR number and repo from context:

```bash
# Get current branch and repo info
git rev-parse --abbrev-ref HEAD
gh repo view --json nameWithOwner -q .nameWithOwner
```

If on a feature branch with an open PR:
```bash
gh pr view --json number,title,url -q '{number: .number, title: .title, url: .url}'
```

If no PR exists yet, use the branch name and commit range as the identifier.

## Step 2: Ask Where to Save

Ask the user where they want to save the review document:

**Options:**
1. **Notion** (Recommended) — Creates a page in Notion via MCP. Searchable, shareable, persistent.
2. **Local Markdown** — Saves to `.reviews/PR-{number}.md` in the repo. Good for git-tracked history.

## Step 3: Build the Document

Structure the document as follows:

```markdown
# Code Review: PR #{number} — {title}

**Date:** {date}
**Branch:** {branch}
**Repo:** {repo}
**Reviewer:** Claude Code (9-agent local review)
**Status:** Pending fixes

## Review Findings

| # | Issue | Sev | Agent | File | Status |
|---|-------|-----|-------|------|--------|
| 1 | {description} | P0 | {agent} | `{file}:{line}` | Open |
| 2 | {description} | P1 | {agent} | `{file}:{line}` | Open |
| ... | ... | ... | ... | ... | ... |

## Detailed Findings

### P0 — Must Fix
{copy all P0 findings with full detail from the review}

### P1 — Should Fix Before Merge
{copy all P1 findings}

### P2 — Should Fix
{copy all P2 findings}

### P3 / Style / Policy
{copy all P3, Style, and Policy findings}

## Review Coverage
{copy the coverage section from the review}

---

## Reviewer Performance Comparison
> This section is populated by `/review:review-measure` after fixes are pushed and PR feedback is collected.

_Run `/review:review-measure` after pushing fixes to populate this section._
```

## Step 4: Save

### Option A: Notion
Use the Notion MCP tools to create a new page:
1. Search for a "Code Reviews" database or parent page: `notion-search` for "Code Reviews" or "Reviews"
2. If not found, ask the user which Notion page to save under
3. Create the page with `notion-create-pages` using the structured content above
4. Return the Notion URL to the user

### Option B: Local Markdown
```bash
mkdir -p .reviews
```
Write the document to `.reviews/PR-{number}.md` (or `.reviews/{branch}.md` if no PR exists).
Add `.reviews/` to `.gitignore` if not already there.

## Step 5: Confirm

```
Review saved to {location}.

Next steps:
1. Fix the issues identified in the review
2. Push your changes to the PR
3. Run /review:review-measure to compare your review against co-worker and bot feedback
```
