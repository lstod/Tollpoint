---
name: tollpoint-review
description: Reviews the current branch's diff against its .tollpoint/<slug>/ spec and plan with the read-only code-reviewer agent, then writes an acceptance criteria checklist (met, unmet, or later) into the PR body, or prints it if there is no PR. Use when the user says review this, tollpoint review, check the ACs, or the build is done and ready for review.
---

# tollpoint-review

The last step of Tollpoint, after you build. Reviews `git diff <base>...HEAD` against the work item's spec and plan, and records the AC checklist where a reviewer will see it: the PR body. It reports findings; fixing them is up to the user.

## Work folder

```bash
common="$(git rev-parse --path-format=absolute --git-common-dir)"
root="$(dirname "$common")"
work="$root/.tollpoint/<slug>"
```

The input is a slug (`/tollpoint-review tag-ideas`). If none is given, use the folder whose slug appears in the branch name, or the only one with an approved plan; otherwise ask. With no work folder at all, review with `none` and skip the checklist.

## Steps

1. **Find the base.** The PR's base if one exists, then the remote's default branch, then `main`:

   ```bash
   git fetch --quiet origin 2>/dev/null
   if b="$(gh pr view --json baseRefName -q .baseRefName 2>/dev/null)"; then base="origin/$b"
   elif b="$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null)"; then base="$b"
   else base="main"; fi
   git diff "$base...HEAD"
   ```

   If `git status --porcelain` shows uncommitted changes, say that they are not in the review.
2. **Dispatch `code-reviewer`** with the base ref, the diff, and the work folder path. Pass nothing else: not the builder's conversation, not a summary of the change.
   - In Claude Code, dispatch it as `tollpoint:code-reviewer`; in Cursor, as `code-reviewer`.
   - If that fails, dispatch a generic read-only subagent with: "Read `~/.cursor/plugins/local/tollpoint/agents/code-reviewer.md` and follow it exactly, with base `<base>` and work folder `<path>`. Do not edit files or run commands." Then give it the diff.
3. **Build the checklist** from the reviewer's per-AC table and the AC text in `spec.md`:

   ```markdown
   ## Acceptance criteria

   Reviewed YYYY-MM-DD by code-reviewer against the local Tollpoint spec `<slug>`.

   - [x] AC-1: <AC text> (met: <evidence>)
   - [ ] AC-2: <AC text> (unmet: <what is missing>)
   - [ ] AC-3: <AC text> (later: <where it is deferred>)
   ```

   Only `met` is checked. Copy each AC's text from the spec, since the spec itself is not on GitHub.
4. **Write it to the PR**, if one exists:

   ```bash
   tmp="$(mktemp)"
   gh pr view --json body -q .body > "$tmp"
   ```

   In `$tmp`, replace the existing `## Acceptance criteria` section (from that heading up to the next `## ` heading or the end) with the new one, or append it if there is none. Leave the rest of the body exactly as it was. Then `gh pr edit --body-file "$tmp"` and remove `$tmp`.
   Without a PR, print the section for the user to paste.
5. **Report** in chat: the reviewer's findings as it returned them, Important first; the checklist; where it was written (the PR link, or "printed, no PR"); and the count of unmet ACs.

## Rules

- Never approve, request changes, or merge, and never resolve a review thread.
- Never fix the findings in this skill. Report them; the user decides what to fix.
- Never change any part of the PR body other than the `## Acceptance criteria` section.

## Done when

- The reviewer's findings and the checklist are in the chat.
- The PR body carries the current `## Acceptance criteria` section, or the user has it to paste.
