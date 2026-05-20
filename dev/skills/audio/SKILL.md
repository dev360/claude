---
name: audio
description: Speak status events aloud via macOS `say` and watch-and-fix PRs after pushing. Trigger when the user types `audio_on` or `audio_off`; when a feature is ready for the user to review; when claude initiates a PR push, sees a PR turn green, or sees auto-merge land; when CI fails on a PR claude just pushed. Also trigger after any `gh pr create` or `git push` that updates an open PR — to start the watch-and-fix loop. Macos-only (requires `/usr/bin/say`).
---

# Audio status announcements + PR watch-and-fix

Two responsibilities, one skill, both gated on a single state file.

## State

`~/.claude/audio.state` — contents are literally `on` or `off`. Treat missing file as `off`. The file is the source of truth; do NOT cache the state in memory across tool calls within a turn — re-read it each time you're about to announce.

- `audio_on` from the user: `echo on > ~/.claude/audio.state` and reply with one line: `audio on.`
- `audio_off` from the user: `echo off > ~/.claude/audio.state` and reply with one line: `audio off.`

If the user says `audio_off`, stop using `say` immediately for the rest of the conversation — even if the state file write fails.

## Announcement rules

Only announce when `cat ~/.claude/audio.state` reads `on`. If it reads `off` or the file is missing, skip silently — never mention the muted announcement in chat.

When announcing, use:

```sh
say "<phrase>"
```

**Strip the `#` from PR numbers before speaking.** macOS `say` pronounces `#` as "inches". Use digits only. `say "PR 42 ready to merge"`, never `say "PR #42 ..."`.

### What to announce

| Event | Phrase |
|---|---|
| A feature/task is finished and the user should look at it | `ready for review` |
| You pushed a PR that the user did NOT explicitly ask you to push in their most recent message | `Pushed up PR <N>` |
| A PR's checks all just turned green and it's mergeable | `PR <N> ready to merge` |
| A PR set to auto-merge actually merged | `PR <N> was merged` |
| You detected CI failure on a PR you just pushed, and are about to fix it | `Fixing PR <N> <problem>` (e.g. `Fixing PR 42 lint errors`, `Fixing PR 42 test failures`, `Fixing PR 42 build error`, `Fixing PR 42 type errors`) |

### "Did the user ask me to push?" rule for `Pushed up PR <N>`

Only announce `Pushed up PR <N>` when **you** decided the push was the next step. If the user's most recent message said "push it", "push this up", "open a PR", "ship it", "create a PR", or similar, skip the push announcement — the user is already watching. Announcements for merge-readiness, merged, and fixing still fire normally.

When in doubt, lean toward NOT announcing — the user knows what they just asked for.

## PR watch-and-fix

Independent of audio state — the fix happens either way; the announcement is the only audio-gated part.

**When to start watching**: any time you push code to a branch with an open PR or create a PR via `gh pr create`. Skip if `gh` is unavailable.

**How**:

1. Get the PR number: `gh pr view --json number -q .number` (run in the repo).
2. Watch in background:
   ```sh
   gh pr checks <N> --watch --fail-fast
   ```
   Use the Bash tool's `run_in_background: true`. Don't poll yourself — `--watch` blocks until checks settle.
3. When it exits non-zero, fetch failure detail:
   ```sh
   gh pr checks <N>
   gh run view <run-id> --log-failed
   ```
4. Classify the failure into one short phrase for the announcement:
   - lint/format → `lint errors`
   - typecheck → `type errors`
   - test failures → `test failures`
   - build failures → `build error`
   - anything else → `CI failure`
5. Announce `Fixing PR <N> <phrase>` (audio-gated). Also write one chat line so the user can see it without audio.
6. Fix the failure. Commit. Push. Restart the watch from step 2.

**Loop limit**: if the same failure class fires three times in a row on the same PR, stop, announce nothing further, and tell the user in chat what's going wrong. Don't burn an evening pushing broken fixes.

**No `--no-verify`**. If a pre-commit hook is failing, that IS the failure to fix.

**Don't watch PRs you didn't push**. If the user pushed manually or asked you to skip CI babysitting, don't start the watch.

## Voice and rate

Default `say` voice is fine. Don't pass `-v` or `-r` unless the user asks.

Phrases are short on purpose — they need to be parseable from across the room. Resist embellishing them.
