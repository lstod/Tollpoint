---
name: tollpoint-plan
description: Turns an approved .tollpoint/<slug>/spec.md into a plan.md listing the files that change and ordered tasks, each tied to acceptance criteria and a verification, plus risks, proof, and rollback, and records Gate 3 when the user approves in chat. Use when the user says write the plan, plan this, how do we build it, move an approved spec to the plan, or approves a plan.
---

# tollpoint-plan

Stage 3 of Tollpoint. Reads `.tollpoint/<slug>/spec.md` (approved) and `intent.md`, and writes `plan.md` in the same folder. Gate 3 (plan approved) belongs to the person in the chat, never to this skill. Nothing here is committed. After Gate 3, build however you like, then run `tollpoint-review`.

## Work folder

```bash
common="$(git rev-parse --path-format=absolute --git-common-dir)"
root="$(dirname "$common")"
work="$root/.tollpoint/<slug>"
```

The input is a slug (`/tollpoint-plan tag-ideas`). If none is given and `$root/.tollpoint/` holds exactly one approved spec with no plan, use it; otherwise ask.

## Rules

- **Approved specs only.** If the spec does not say `Status: approved`, stop. Write nothing, and reply: "Gate 2 has not passed for <slug> (spec status `<status>`), so no plan can be written." Do not offer to approve it.
- **No open questions.** If the spec's `## Open questions` holds anything other than `None stated.`, stop. Write nothing, and reply: "Spec <slug> has unresolved open questions, so no plan can be written:", then list them. Do not answer them yourself.
- **Explore read-only.** Do not edit, create, or run anything that changes state until the plan file is written.
- **Every AC maps to a task.** Each `AC-n` in the spec appears in at least one task's `ACs:` field, and no task cites an AC the spec does not have.
- **Every task has a verification.** `Verify:` is either one command in backticks that can run when the task is done, or `manual:` followed by what a human checks. Never write a placeholder command.
- **Small tasks.** `Size` is `S` (under an hour) or `M` (one sitting). Split anything larger.
- **Flag scope creep.** Every row in Files that change names the AC that needs it. A file no AC needs is still listed, with a `Why` that starts `scope creep:` and says why it is still needed. Never leave a needed file out to keep the count down; the review checks every changed path against this table.
- **The spec is the contract.** If the plan cannot meet an AC as written, stop and say so; the spec gets a revision. Do not bend the plan around the spec.

## Steps

1. **Find the work.** Read `$work/spec.md` and check the Rules. Read `$work/intent.md` for `Risk:`. If `plan.md` already exists: if it is `approved`, stop and ask whether to add a revision; if it is `draft`, update it.
2. **Read the spec in full:** Task, Context, Constraints, Format, Acceptance criteria, Non-goals, Risks, Verification, and the Revision log.
3. **Explore read-only.** Open every file the spec's Context cites and every file you expect to change. Note existing scripts, tests, and conventions a task can reuse as its verification.
4. **List the files that change.** One row per file: path, change (`add`, `edit`, `delete`, or `rename`), and the ACs that need it, or `scope creep: <why>`.
5. **Write the tasks.** Order them so each task depends only on earlier ones. One line each:

   ```
   - T1: <description> | ACs: AC-1, AC-2 | Verify: `<command>` | Size: S | Depends on: none
   - T2: <description> | ACs: AC-3 | Verify: manual: <what a human checks> | Size: M | Depends on: T1
   ```

   Number `T1..n` with no gaps. `Depends on` is `none` or earlier task IDs. Write a pipe inside a field as `\|`.
6. **Fill in the rest** of the template below, keeping every heading in order and removing the guidance lines.
7. **Check coverage yourself:** list every `AC-n` in the spec and confirm each appears in a task's `ACs:` field. Fix any gap before reporting.
8. **Report** to the user: the file path, the number of tasks and files, any `scope creep:` rows, any `manual:` verifications, and the next step: Gate 3.

## Template

```markdown
# Plan: <short title>

- Spec: [spec.md](spec.md)
- Spec approved: YYYY-MM-DD
- Status: draft

## Files that change
Every Why names an AC-n or starts with "scope creep:".
| Path | Change | Why |
|---|---|---|
| `path/to/file` | add | AC-1 |

## Tasks
One line per task, numbered T1..n. Verify is a command in backticks, or "manual:" and what a human checks. Size is S or M. Depends on is "none" or earlier task IDs.
- T1: <description> | ACs: AC-1 | Verify: `<command>` | Size: S | Depends on: none

## Parallel groups
Tasks with no dependency between them.

## Checkpoints
Where to stop, verify, and confirm before continuing.

## Risks and mitigations

## Proof
Commands and expected results that show the spec is met.

## Rollback
How to undo this change if it ships and misbehaves.

## Revision log
Gate 3 approval is the first line.
```

`Spec approved:` is the date on the spec's Gate 2 line.

## Gate 3

The user reviews the plan in chat and approves it. When they say so explicitly, in one change:

- `- Status: approved`
- a first `## Revision log` line: `- YYYY-MM-DD: Gate 3 approved by <name> in chat.`

Silence, "carry on", or starting to build from the plan is not approval. If the spec has gone back to `draft` since the plan was written, approval waits for Gate 2 again.

## Done when

- `$work/plan.md` exists with every template heading.
- Every AC of the spec is covered by a task, and every task has a `Verify:`.
- The report lists any scope creep and manual verifications, and Gate 3 is either recorded or waiting for the user.
