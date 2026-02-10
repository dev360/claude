---
description: Push branch and create a PR with filled-out template
argument-hint: <optional: Linear ticket ID or GH issue ID, e.g. FLO-1111 or GH-42>
---

# Create Pull Request

You are a PR creation assistant. Follow these steps exactly.

## Step 0: Gate — Require Code Review

Before doing ANYTHING else, check whether `/review:review` has been run in this session.

**How to check:** Look back through the conversation history for a "Code Review Summary" output from the review orchestrator.

- If a review HAS been run → proceed to Step 1.
- If a review has NOT been run → check the diff size:
  - If the change is **trivially small** (1-3 lines changed, single file, obvious fix like a typo/config tweak) → proceed with a note: _"Skipping review gate — change is trivial."_
  - Otherwise → **STOP** and tell the user:
    > You haven't run `/review:review` yet. Please run it before creating a PR. This ensures code quality issues are caught before the PR is opened.
    >
    > If this is a trivial change, let me know and I'll skip this check.

## Step 1: Check Working Tree

```bash
git status
```

- If there are **uncommitted changes** (staged or unstaged) → **STOP** and ask the user to commit first:
  > You have uncommitted changes. Please commit them before creating a PR, or ask me to `/commit` for you.
- If the working tree is clean → proceed.

## Step 2: Push the Branch

```bash
# Get current branch name
git rev-parse --abbrev-ref HEAD
```

- If on `main` or `master` → **STOP** and tell the user to create a feature branch first.
- Otherwise, push:

```bash
git push -u origin HEAD
```

## Step 3: Create the PR with Default Template

Create the PR using `gh`, letting the repo's PR template populate the body. Do NOT fill in any body content yet.

```bash
gh pr create --fill --draft
```

If the repo has no PR template, create with just the title (you'll fill the body in Step 5).

**IMPORTANT:** If `gh pr create` fails because a PR already exists, just use the existing PR.

## Step 4: Determine the PR Title

The title MUST follow this format:

```
[TICKET-ID] - Short description of the change
```

Examples:
- `[FLO-1111] - Fix broken landing page`
- `[GH-42] - Add user avatar to profile header`
- `[ENG-305] - Migrate auth to OAuth2`

**To find the ticket ID:**
1. If the user provided one as an argument → use it.
2. Check the branch name for a ticket pattern (e.g., `flo-1234-fix-thing`, `gh-42-add-feature`).
3. Check recent commit messages for ticket references.
4. If no ticket ID can be found → ask the user:
   > What's the Linear ticket or GitHub issue ID for this change? (e.g., FLO-1111, GH-42)

**To write the description:**
- Derive it from the diff. Be succinct — aim for 5-10 words max.
- Use imperative mood ("Fix", "Add", "Update", "Remove", not "Fixed", "Adding").

Update the PR title:

```bash
gh pr edit <PR_NUMBER> --title "[TICKET-ID] - Description"
```

## Step 5: Fill Out the PR Template

Now pull down the PR body that was created (which should contain the repo's PR template):

```bash
gh pr view <PR_NUMBER> --json body --jq '.body'
```

Also gather the full diff for context:

```bash
git diff main...HEAD
```

Now fill out the PR template:
- **Be succinct.** One-liners where possible. No essays.
- Fill in every section the template asks for.
- If the template has checkboxes, check the ones that apply.
- If there's a "Screenshots" section and the change is not visual, write "N/A".
- If there's a "Testing" section, briefly describe how to verify the change.
- Reference the ticket/issue ID where appropriate.

Update the PR body:

```bash
gh pr edit <PR_NUMBER> --body "$(cat <<'EOF'
<filled template content>
EOF
)"
```

## Step 6: Confirm

Print the PR URL and a brief summary:

```
PR created: <URL>
Title: [TICKET-ID] - Description
Status: Draft
```

Ask the user if they want to mark it as ready for review (`gh pr ready`).
