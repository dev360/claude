---
description: Pull down CI/CD comments from a PR and fix them
argument-hint: <optional: PR number>
---

# Fix PR Comments from CI/CD

You are a PR comment fixer. You pull down CI/CD feedback from a PR and resolve each item.

## Step 0: Gate — Verify PR Exists and CI is Complete

First, determine the PR number:

```bash
# Get current branch
git rev-parse --abbrev-ref HEAD
```

```bash
# Find PR for this branch
gh pr view --json number,state,statusCheckRollup --jq '{number, state, checks: [.statusCheckRollup[] | {name: .name, status: .status, conclusion: .conclusion}]}'
```

**If no PR exists for this branch → STOP:**
> No PR found for this branch. Create one first with `/dev:pr`.

**If CI checks are still running (any check has `status: "IN_PROGRESS"` or `status: "QUEUED"`) → STOP:**
> CI is still running. Wait for it to finish before fixing comments. Currently running:
> - [list running checks]

**If CI checks have all completed → proceed.**

## Step 1: Gather All PR Comments

Pull down all review comments, bot comments, and check annotations:

```bash
# Get review comments (from humans and bots)
gh api repos/{owner}/{repo}/pulls/{PR_NUMBER}/comments --jq '.[] | {id, user: .user.login, body: .body, path: .path, line: .original_line, created_at: .created_at}'
```

```bash
# Get issue-level comments (general PR comments, often from CI bots)
gh api repos/{owner}/{repo}/issues/{PR_NUMBER}/comments --jq '.[] | {id, user: .user.login, body: .body, created_at: .created_at}'
```

```bash
# Get check run annotations (lint errors, test failures, etc.)
gh api repos/{owner}/{repo}/commits/$(git rev-parse HEAD)/check-runs --jq '.check_runs[] | select(.conclusion == "failure" or .conclusion == "action_required") | {name: .name, output_title: .output.title, annotations: [.output.annotations[]? | {path: .path, start_line: .start_line, end_line: .end_line, message: .message, annotation_level: .annotation_level}]}'
```

**If there are NO comments and all checks passed → STOP:**
> All CI checks passed and there are no review comments. Nothing to fix!

## Step 2: Categorize and Present

Group all feedback items into:

1. **CI/Lint errors** — from check annotations and bot comments (ESLint, TypeScript, Prettier, etc.)
2. **Test failures** — from check runs or bot comments about failing tests
3. **Review comments** — from human reviewers or review bots (e.g., CodeRabbit, Copilot)
4. **Other** — anything else

Present them to the user as a numbered list:

```
Found N items to address:

### CI/Lint Errors
1. `src/foo.ts:42` — ESLint: no-unused-vars (variable `bar` is declared but never used)
2. `src/baz.ts:10` — TypeScript: Type 'string' is not assignable to type 'number'

### Test Failures
3. `tests/auth.test.ts` — "should redirect on expired token" — Expected 302, got 200

### Review Comments
4. `src/api/handler.ts:55` — @reviewer: "This should handle the 404 case"
```

## Step 3: Fix Each Item

Work through the items one by one:

- **For clear, unambiguous fixes** (lint errors, type errors, formatting) → fix them directly.
- **For ambiguous items** (review comment that could be interpreted multiple ways, test failure with unclear root cause) → **ASK the user** before making changes:
  > Item #4: @reviewer says "This should handle the 404 case". I see two options:
  > 1. Add a 404 check before the DB call
  > 2. Add a try/catch around the fetch and return 404 from the catch
  > Which approach do you prefer?
- **For items you cannot fix** (e.g., requires infra changes, CI config issues) → flag them:
  > Item #5: This is a CI configuration issue (flaky test timeout). I can't fix this from code. You may need to update the CI config or re-run the pipeline.

Read the relevant files before making any changes. Understand the context.

## Step 4: Verify

After fixing all items:

```bash
# Run the same checks locally if possible (lint, typecheck, tests)
# Adapt these to the project's actual commands
npm run lint 2>&1 || yarn lint 2>&1 || true
npm run typecheck 2>&1 || yarn typecheck 2>&1 || true
npm test 2>&1 || yarn test 2>&1 || true
```

Report results. If everything passes, tell the user. If not, continue fixing.

## Step 5: Summary

Once all items are resolved:

```
Fixed N/M items:
- [x] #1 — Fixed unused variable in src/foo.ts
- [x] #2 — Fixed type error in src/baz.ts
- [x] #3 — Fixed test assertion in tests/auth.test.ts
- [x] #4 — Added 404 handling per reviewer request
- [ ] #5 — CI config issue (requires manual intervention)
```

Ask the user if they want to commit and push the fixes.
