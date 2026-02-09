---
description: Compare local review performance against PR feedback from co-workers and CI bots
argument-hint: (optional) PR number or URL
---

# Review Measure

You compare the findings from a saved `/review:review` against actual PR feedback — co-worker comments, CI bot comments, and automated review tools. The output is a structured performance comparison.

## Step 0: Context Gate

You need a saved review to compare against. Check for it:

1. **In-context**: Look for review findings from this conversation (numbered findings with P0-P3 severities).
2. **Local file**: Check `.reviews/` directory for a matching PR file.
3. **Notion**: If the user provides a Notion URL, fetch the review from there.

If NONE of these exist:
```
ERROR: No saved review found.

Run /review:review first, then /review:review-save to persist it.
Cannot measure without a baseline.
```
**Stop here.**

## Step 1: Identify the PR

```bash
gh pr view --json number,title,url,headRefName -q '{number: .number, title: .title, url: .url, branch: .headRefName}'
```

If no argument provided and no open PR on current branch, ask the user for the PR number.

## Step 2: Pull All PR Comments

Fetch every comment and review from the PR:

```bash
# PR review comments (inline code comments)
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate

# PR reviews (top-level review bodies)
gh api repos/{owner}/{repo}/pulls/{number}/reviews --paginate

# Issue comments (general PR conversation)
gh api repos/{owner}/{repo}/issues/{number}/comments --paginate
```

## Step 3: Classify Reviewers

Group all comments by author. Classify each author:

| Type | Detection |
|------|-----------|
| **Co-worker** | `user.type == "User"` and not a known bot name |
| **Bot** | `user.type == "Bot"` OR name ends in `[bot]` OR known bot names |

For bots, use the actual bot name as the column header (e.g., "Cursor Bot", "CodeRabbit", "Claude Bot", "GitLab Duo", "SonarQube", etc.).

For co-workers, use their GitHub username.

**Important**: The column names are dynamic — they depend on who actually commented on the PR. Do not hardcode column names.

## Step 4: Extract Findings per Reviewer

For each reviewer, extract their distinct findings:
- Parse each comment for the **issue being raised** (ignore "LGTM", approval comments, CI status updates that don't flag issues)
- Normalize each finding to a short description
- Assign a severity if the reviewer indicated one, otherwise estimate based on the issue type
- Note which file/line the comment targets (if inline)

## Step 5: Build the Comparison Matrix

Cross-reference all findings:

1. Start with the local review findings as rows (from the saved review)
2. For each external finding, check if it matches an existing local finding (same file, same issue, even if described differently)
3. If an external finding is NEW (not in local review), add it as a new row
4. Mark each cell: checkmark if that reviewer found it, blank if they missed it

### Output Table

```markdown
## Reviewer Performance Comparison

Comparison of issues found by each reviewer on PR #{number}.
**Local Review** = Claude Code (9-agent review), **{reviewer_name}** = {description}.

| Issue | Sev | Local Review | {Reviewer 1} | {Reviewer 2} | ... |
| --- | --- | --- | --- | --- | --- |
| #1 {description} | P0 | {check} | {check} | {check} | ... |
| #2 {description} | P1 | {check} | {check} | {check} | ... |
| ... | ... | ... | ... | ... | ... |
| {new external finding} | {sev} |  | {check} |  | ... |

> Use checkmark for found, blank for missed. Order rows by severity (P0 first, then P1, P2, P3, Style, Policy).

### Summary

| Reviewer | Issues Found | Exclusive Finds |
| --- | --- | --- |
| **Local Review** (Claude Code, 9 agents) | {count} | {count} ({list if few}) |
| **{Reviewer}** ({type}) | {count} | {count} ({list if few}) |

### Takeaways

Write 3-5 bullet points analyzing:
- What the local review caught that others missed
- What co-workers or bots caught that the local review missed (blind spots)
- Which bot was most/least effective
- Any patterns in what was missed (e.g., "design system knowledge", "org-specific policy")
```

## Step 6: Update the Saved Document

If the review was saved to Notion, update the existing page with the comparison section.
If saved locally, append the comparison section to the `.reviews/PR-{number}.md` file.

## Step 7: Report

```
Review measurement complete for PR #{number}.

Local Review: {found}/{total} issues ({exclusive} exclusive)
{Reviewer}: {found}/{total} issues ({exclusive} exclusive)
...

Blind spots: {list any issues only found externally}

{link to updated document if applicable}
```
