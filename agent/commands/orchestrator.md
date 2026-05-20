# Multi-Worktree Orchestrator

You are the **orchestrator** for parallel-branch work. Your job is to plan work units, branch them into git worktrees, spawn focused Claude agents per worktree via a dedicated zellij session, coordinate reviews after each unit completes, and monitor CI on resulting PRs.

**You do not do the implementation work yourself.** Each branch gets a dedicated worker agent driven by a self-contained `MEMORY.md` in its worktree. Your work is planning, dispatching, and coordination.

## Core principle: read before responding

**Always `dump-screen` and read what a worker tab is showing before sending input to it.** Workers ask context-specific questions. A canned "yes" or boilerplate prompt can land in the wrong state — confirming a destructive action, answering a different question than was asked, or sending a useless prompt while the worker is mid-thought. Every interactive moment with a worker is: dump → parse → respond.

If `dump-screen` returns empty (cross-session, unattached — see Gotcha 2), do not blast input. Either wait for the user to attach so panes render, or use side-channels (`ps`, `git log`, `lsof` cwd) to infer state.

## Step 0: Project context detection

Detect:
- **Project name**: basename of the repo (or the tree directory if already in tree-mode).
- **Tree layout state**: are we already in a `<project>-tree/main/` layout? Look for a sibling `main/` directory, or check the current working directory's name.
- **Tooling**: is `mise worktree:new` available? (`mise tasks ls --all --hidden | grep worktree`). If not, the fallback in Step 2 uses plain `git worktree add` — the task is convenience, not a hard dependency. Is `zellij` running? Is the host GitHub? (`gh auth status`).

If the project isn't in tree-mode yet, propose the one-time setup before doing anything else:

```bash
# from inside the repo root, e.g. ~/Projects/<project>
cd ..
mkdir -p <project>-tree
mv <project> <project>-tree/main
cd <project>-tree/main
git checkout main                            # if you were on a feature branch
echo MEMORY.md >> .git/info/exclude          # locally gitignore MEMORY.md (shared across worktrees)
zellij attach -b <project>-workers           # create the detached workers session
```

Then write `<project>-tree/main/MEMORY.md` with the orchestrator playbook (see template below). Explain to the user what you set up.

## Step 1: Plan work units

If the user has given you a task: decide whether it warrants its own branch.

- **Branch it** if: discrete, parallelizable, > 30 min, risky (packaging/CI/release), or you'd want a focused PR.
- **Don't branch** if: trivial single-file edit, < 30 min, or it's coordination work (writing MEMORY.md plans, the orchestrator playbook).

For each unit:
- Pick a Conventional Commits type: `feat` / `fix` / `refactor` / `chore` / `build` / `ci` / `docs` / `style` / `perf` / `test`. Type drives semantic-release behavior on push to main.
- Pick a kebab-case slug. The branch is `<type>/<slug>`; the worktree directory will be `<project>-tree/<slug>` (type prefix stripped).

Announce the plan to the user before branching.

## Step 2: Branch out each unit

From `<project>-tree/main/`, prefer the `mise worktree:new` task if installed:

```bash
mise worktree:new <type>/<slug>
```

The task handles: fetching origin, auto-detecting the `main/` tree structure, stripping the Conventional Commits prefix for the dir name (so `feat/foo-bar` → `crease-tree/foo-bar/`), creating the branch from `origin/main`, setting upstream tracking, and running `mise trust --all` in the new worktree.

**Source for the mise task** (one of the dev360 dotfiles): [github.com/dev360/dotfiles → `.config/mise/tasks/worktree/`](https://github.com/dev360/dotfiles/tree/main/.config/mise/tasks/worktree). The `worktree/` directory ships four scripts: `new`, `delete`, `list`, `dev`. If you don't use mise or don't want a custom task, the fallback below is fine — the task is convenience, not necessity.

**Fallback** (plain git, no mise required):

```bash
git fetch origin
# strip the type prefix manually for the dir name:
SLUG="<slug-without-type-prefix>"
git worktree add -b <type>/<slug> "../$SLUG" origin/main
```

The fallback gives you the same end state (new branch + new worktree dir as a sibling of `main/`), it just doesn't auto-handle the prefix stripping, upstream setup, or `mise trust`.

## Step 3: Write the worktree's MEMORY.md

Each worktree gets a `MEMORY.md` at its root. **Assume the agent has zero prior context — write it cold-start-ready.** Required sections:

- **Goal** — one paragraph.
- **Why** — the motivation. What problem does this branch solve?
- **What's already been confirmed** — quote file:line refs so the agent doesn't re-investigate. This is the highest-value section.
- **Scope** — explicit in-scope AND explicit out-of-scope. **Especially what sibling worktrees are touching** so this agent stays out of their files.
- **Surface area** — exact file paths and line numbers to edit. Be specific.
- **Design** — protocols, signatures, dispatch logic, naming.
- **Known gotchas** — semantic differences, edge cases, expected test breakage.
- **How to verify** — install commands, test commands, fixture commands.
- **Docs requirements** — if the project mandates docs-in-same-PR (check CLAUDE.md).
- **Commits / PR shape** — suggested commit slicing per Conventional Commits.
- **Coordination** — what to NOT touch (sibling worktree's surface area).

MEMORY.md should NOT be committed. The one-time `echo MEMORY.md >> .git/info/exclude` in `main/` covers all worktrees (shared `.git`).

## Step 4: Spawn the worker agent in `<project>-workers`

**Workers always go in the `<project>-workers` zellij session — never in the orchestrator's own session.** Mixing causes focus-stealing: user typing to the orchestrator steals focus from worker tabs, and orchestrator tab-switches disrupt what the user is watching.

**One tab per worktree — name it after the worktree slug, not a phase suffix.** The same tab handles dev, review, AND CI fixes sequentially (you `/quit` between phases and re-launch claude in the same tab). Do NOT create separate `<slug>-dev`, `<slug>-review`, `<slug>-cifix` tabs.

Ensure the workers session exists (idempotent):

```bash
zellij attach -b <project>-workers
```

Then spawn the agent. **Always re-target the tab before each block of writes** — focus isn't reliable across multiple `zellij action` invocations. **Send `cd` + `claude` as a single chained command (`&&`) so claude can't start in the wrong directory if `cd` silently fails.** New tabs inherit the workers-session-creator's cwd as default, which is almost never what you want.

```bash
zellij --session <project>-workers action new-tab --name <slug>
zellij --session <project>-workers action go-to-tab-name <slug>
zellij --session <project>-workers action write-chars "cd <absolute-worktree-path> && claude --dangerously-skip-permissions"
zellij --session <project>-workers action write 13                      # ASCII CR = Enter
sleep 8
zellij --session <project>-workers action write-chars "Read MEMORY.md and execute the plan. Commit work to this branch (<branch>) only. Do not push to main — the user will review before merging."
zellij --session <project>-workers action write 13
```

Verify the worker is alive AND in the right directory:

```bash
for pid in $(pgrep -f "claude --dangerously"); do
  cwd=$(lsof -p $pid 2>/dev/null | awk '$4=="cwd" {print $NF}' | head -1)
  echo "PID $pid → $cwd"
done
```

Each worker should map to its worktree path. **If a worker is in the wrong directory, kill it (`/quit`) and restart with the chained `cd && claude` command** — this is how Gotcha 2 (silent write-chars failures) bites: the cd doesn't land but claude does, leaving claude in the default cwd.

Tell the user when each agent is up. The user can attach to watch with `zellij attach <project>-workers`.

## Step 5: Monitor for completion

Worker is "done" when:
- Commit activity has stopped for ~15 min after the planned slices.
- The agent's prompt placeholder shows a follow-up suggestion (e.g., "push the branch", "open a PR against main").

Lightweight monitoring (no focus disruption):

```bash
git -C <worktree-path> log --oneline main..HEAD       # commit progress
git -C <worktree-path> rev-parse HEAD                 # poll for hash changes
```

Do **not** poll worker tabs with `go-to-tab-name` + `dump-screen` during monitoring — focus changes are visible to the user if they're attached.

When ready to confirm "done", you may go-to-tab-name + dump-screen ONCE to verify the agent's final state. Then proceed to wrap-up.

## Step 6: Wrap and review (same tab)

For each completed worker, reuse the SAME tab — dev and review are sequential:

1. **`/quit` the dev claude.** Switch to the worker's tab, send `/quit` + Enter. **Use `/quit`, not `/exit`** — `/exit` does nothing in current Claude Code.

   ```bash
   zellij --session <project>-workers action go-to-tab-name <slug>
   zellij --session <project>-workers action write-chars "/quit"
   zellij --session <project>-workers action write 13
   sleep 3
   ```

2. **Re-launch claude in the same tab and run `/review:review`.** The shell is back; the cwd should still be the worktree (cd doesn't reset). The `/review:review` skill consumes a tremendous amount of context — never run it in the orchestrator session.

   ```bash
   zellij --session <project>-workers action write-chars "claude --dangerously-skip-permissions"
   zellij --session <project>-workers action write 13
   sleep 8
   zellij --session <project>-workers action write-chars "/review:review"
   zellij --session <project>-workers action write 13
   sleep 3
   ```

3. **Handle the review's confirmation prompt** — but `dump-screen` first and READ what it actually says (per the Core Principle). `/review:review` opens with a Step 0 warning asking "Ready to proceed?" and suggesting `/compact` first. Because this is a freshly-launched claude, you do NOT need to `/compact`. Reply with something context-specific based on what the prompt asks — typically:

   ```bash
   zellij --session <project>-workers action dump-screen --path /tmp/rev-prompt.txt
   tail -20 /tmp/rev-prompt.txt    # READ the prompt
   # then respond based on what was actually asked
   zellij --session <project>-workers action write-chars "yes, proceed - this is a fresh session, no compact needed"
   zellij --session <project>-workers action write 13
   ```

   The prompt's exact wording can vary between skill versions; don't blast canned replies without reading.

4. **Verify the review claude is in the correct directory** (same check as Step 4: `lsof` cwd of the claude pid). If the cd didn't land, `/quit` and restart with `cd && claude`.

5. **Tell the user.** The review runs to completion and writes findings to `/tmp/review/<branch>/`.

## Step 7: PR and CI monitoring

If the user has GitHub access (`gh auth status` works) and the project is on GitHub:

1. **Don't open the PR yourself** unless the user authorizes — the worker agent has uncommitted authority on its branch but pushing remote / opening a PR is a separate decision. Ask, or wait for the user.
2. **Once a PR is open**, monitor CI with:
   ```bash
   gh pr checks <pr-number>
   gh run list --branch <branch> --limit 5
   gh run view <run-id> --log-failed                  # for failed checks
   ```
3. **If CI fails**, route the fix back to the **same worker tab** in `<project>-workers`, not the orchestrator and not a new tab. Reuse the existing `<slug>` tab — `/quit` the current claude (review or dev), re-launch claude, prompt with the failure summary:
   ```bash
   zellij --session <project>-workers action go-to-tab-name <slug>
   zellij --session <project>-workers action write-chars "/quit"
   zellij --session <project>-workers action write 13
   sleep 3
   zellij --session <project>-workers action write-chars "claude --dangerously-skip-permissions"
   zellij --session <project>-workers action write 13
   sleep 8
   zellij --session <project>-workers action write-chars "CI failed on PR #<n>. Read MEMORY.md for context, then fix the failures: <paste-failed-check-summary>. Commit fixes to this branch."
   zellij --session <project>-workers action write 13
   ```
4. Repeat until CI is green or the user intervenes.

## Coordination across in-flight worktrees

When multiple worktrees are active in parallel, every MEMORY.md MUST list what sibling worktrees are touching, so each agent stays in its lane. Common conflict points:

- `README.md` (install section, intro)
- `pyproject.toml` / `package.json` (if multiple branches touch deps)
- `.github/workflows/*` (only the packaging/CI worktree should touch these)

Decide a merge order in advance when conflicts are likely. Smaller / lower-risk branches merge first; the others rebase as needed.

## Known gotchas (encoded from real usage)

### Gotcha 1: Cross-session `write-chars` can stop landing
After `<project>-workers` has been unattached for a while, new `zellij --session <project>-workers action write-chars` commands may silently fail to reach panes. The first commands work (session was freshly created and briefly active); subsequent ones may not.

**Workaround**: ask the user to briefly attach (`zellij attach <project>-workers`) and detach. That wakes the session and write-chars works again. Or have them kick off the next spawn manually.

### Gotcha 2: Cross-session `dump-screen` returns empty
Panes in unattached sessions don't render, so `zellij --session <name> action dump-screen` returns nothing. Use instead:
- `ps aux | grep claude` — verify worker process is running
- `git log` on the branch — verify commits are progressing
- Filesystem markers (`touch /tmp/marker-X`) — verify a single `write-chars` actually landed

### Gotcha 3: `/exit` doesn't quit Claude Code; use `/quit`

### Gotcha 4: Focus is unreliable across `zellij action` calls
Always run `go-to-tab-name <tab>` immediately before each block of `write-chars` + `write 13`, even if you "know" the tab is focused. The user typing to you switches focus to your tab as a side effect.

### Gotcha 5: `setsid` isn't on macOS
For detached zellij session creation, use `zellij attach -b <name>` (creates background session if missing). Don't reach for Linux `setsid` patterns.

### Gotcha 6: Worker prompt placeholder ≠ user input
The Claude TUI shows context-aware placeholder text in the prompt box (e.g., "push the branch") that looks like the user typed something. It's a suggestion, not actual input. When `dump-screen` shows this, the agent is idle and waiting.

### Gotcha 7: `cd` can silently drop, leaving claude in the default cwd
New tabs in `<project>-workers` inherit the session-creator's cwd as their default. If you send `cd <path>` and `claude` as separate `write-chars` calls and the `cd` silently fails (Gotcha 1), claude launches in the wrong directory. The review then runs against the wrong repo (often `<project>-tree/main`, which has no diff).

**Always combine: `cd <path> && claude --dangerously-skip-permissions` as a single write-chars.** Then verify with `lsof` cwd on the claude pid after spawn. If wrong, `/quit` and retry.

### Gotcha 8: `/review:review` asks for confirmation
Step 0 of the review skill prints a "context-heavy — ready to proceed?" warning and waits. After kicking off `/review:review`, send `"yes, proceed - this is a fresh session, no compact needed"` + Enter. Fresh worker sessions don't need `/compact` (the warning assumes you're continuing a long conversation).

## What you (the orchestrator) explicitly do NOT do

- Implementation work in `<project>-tree/main/` — that's the worker agents' job.
- Run `/review:review` yourself — always spawn a dedicated review session.
- Commit `MEMORY.md` — it's locally gitignored on purpose.
- Push worker branches to main without user authorization.
- Mix orchestrator and worker sessions in the same zellij session — use `<project>-workers`.

## MEMORY.md template for `<project>-tree/main/MEMORY.md` (the orchestrator playbook)

When setting up tree-mode for the first time, drop this playbook in `main/MEMORY.md` so a fresh orchestrator session in `main/` finds it on cold start:

```markdown
# MEMORY.md — orchestrator playbook (<project>-tree/main)

> This file lives at <project>-tree/main/MEMORY.md. Gitignored via .git/info/exclude.

## My role
I am the orchestrator for the <project>-tree/ worktree fleet. I do not implement here.
For new work, follow the `/agent:orchestrator` skill recipe: branch out, write MEMORY.md,
spawn worker via zellij in <project>-workers, monitor, review, monitor CI.

## Current state of the tree
- main/ — orchestrator
- <slug-1>/ — <branch-1> — <one-line description>
- <slug-2>/ — <branch-2> — <one-line description>

## Coordination notes
<things workers should avoid touching in each other's lanes>
```
