---
name: tollpoint-spec
description: Turns an accepted intent into a verifiable .tollpoint/<slug>/spec.md with numbered acceptance criteria, hardens it with the spec-critic agent, records the critique in the spec's Critique section, and records Gate 2 when the user approves in chat. Use when the user says write the spec, design this, move an accepted intent to the spec, or approves a spec.
---

# tollpoint-spec

Stage 2 of Tollpoint. Reads `.tollpoint/<slug>/intent.md` (accepted) and writes `spec.md` in the same folder. Gate 2 (spec approved) belongs to the person in the chat, never to this skill. Nothing here is committed.

## Work folder

```bash
common="$(git rev-parse --path-format=absolute --git-common-dir)"
root="$(dirname "$common")"
work="$root/.tollpoint/<slug>"
```

The input is a slug (`/tollpoint-spec tag-ideas`). If none is given and `$root/.tollpoint/` holds exactly one accepted intent with no spec, use it; otherwise ask.

## Rules

- **Accepted intents only.** If the intent's `Status:` is not `accepted`, stop. Write nothing, and reply: "Gate 1 has not passed for <slug> (status `<status>`), so no spec can be written." Do not offer to accept it.
- **Ground every Context claim in a file.** Cite the path (and line numbers where they help) and state what the file does today. A claim you cannot cite does not go in.
- **Never resolve a product question silently.** If the intent and the code leave a choice open that changes what users get, write it under `## Open questions` with the options and a suggested default. Technical choices you can justify go under `## Design decisions` with the alternatives.
- **Every AC is testable.** Each `AC-n` names something a command, a test, or an artifact can show. "Works well" or "is clear" is not an AC.
- **The spec is the contract.** If an implementation violates the spec, fix the spec and regenerate; never bend the code around the spec.
- **Changes after approval** go in `## Revision log`, one dated line each, with the reason. A change to an acceptance criterion, the Format, or the Non-goals also sets `Status: draft` until the user approves again. A correction that changes none of these (a citation, a typo) keeps `approved`.

## Steps

1. **Find the work.** Read `$work/intent.md` and check `Status: accepted`. If `spec.md` already exists: if it is `approved`, stop and ask whether to add a revision; if it is `draft`, update it rather than starting over.
2. **Read the intent in full**, including any `## Revision` sections (they override the text above them).
3. **Explore read-only.** Open the files the work will touch and the files the intent names. Note current behavior, existing conventions, and anything that contradicts the intent.
4. **Draft `$work/spec.md`** from the template below:
   - Line 1 `# Spec: <title>` (the intent's title unless a shorter one is clearer), `- Intent: [intent.md](intent.md)`, `- Status: draft`.
   - Keep every heading, in template order. Remove the guidance lines. A section with nothing to say gets `None stated.`.
   - `## 5. Example`: at least one concrete input and its expected output, plus one failure or refusal case where one exists.
   - `## Acceptance criteria`: `- AC-1:` to `- AC-n:`, numbered with no gaps.
   - `## Verification`: how each AC is proved. Every AC appears.
   - `## Revision log`: empty until Gate 2.
5. **Critique** (see "Critique").
6. **Revise.** For each finding, decide:
   - `accepted:` change the spec, and say what changed (for example "added AC-7", "rewrote Context bullet 3").
   - `declined:` say why (out of scope with the non-goal that covers it, or wrong with the evidence).
   - `open question:` the finding needs a product decision; add it to `## Open questions` with options and a suggested default.
   - Every `blocking` finding is `accepted:` or `open question:`. Never decline a blocking finding on your own judgment. Only the user can, recorded as `declined: decided by <name> ...`.
7. **Write `## Critique`** at the end of the spec (see "Critique section").
8. **Report** to the user: the file path, the number of ACs, the critic's findings by severity and how many changed the spec, every open question, and the next step: Gate 2.

## Template

```markdown
# Spec: <short title>

- Intent: [intent.md](intent.md)
- Status: draft

## 1. Task
What will be built, in one paragraph.

## 2. Context
Grounded in real files: cite paths and current behavior.

## 3. Constraints
Technical, policy, compatibility, performance.

## 4. Format
Interfaces, contracts, data shapes, file formats.

## 5. Example
At least one concrete input and expected output.

## Acceptance criteria
- AC-1: <observable, testable statement>

## Non-goals

## Risks

## Design decisions
Decision, alternatives considered, why this one.

## Verification
How each AC will be proved (test, command, screenshot).

## Open questions
Questions the user must decide before approval. Never resolved silently.

## Revision log
Changes after approval, with reason. Gate 2 approval is the first line.

## Critique
```

## Critique

Dispatch the read-only `spec-critic` agent with only the spec's path and the instruction "Critique this spec". Do not summarize the spec or say what you think of it.

- In Claude Code, dispatch it as `tollpoint:spec-critic`; in Cursor, as `spec-critic`.
- If that fails, dispatch a generic read-only subagent with: "Read `~/.cursor/plugins/local/tollpoint/agents/spec-critic.md` and follow it exactly on `<path>`. Do not edit files or run commands." Record this as `spec-critic (<harness>, stand-in)`.
- One pass per draft. Do not re-run the critic until it agrees.

## Critique section

The last section of `spec.md`. Replace it on each new pass:

```markdown
## Critique

- Critic: spec-critic (<harness>[, stand-in])
- Date: YYYY-MM-DD

| # | Kind | Severity | Finding | Evidence | Disposition |
|---|---|---|---|---|---|
| 1 | weakness | important | ... | `path:line` | accepted: ... |

No blocking findings.
```

Copy the critic's rows verbatim, then add the Disposition. The last line is `No blocking findings.` if no row is `blocking`, or `Blocking findings resolved.` if at least one is and each is accepted or an open question. If the critic could not be dispatched, the section holds only `Critic: not run (<reason>).`

## Gate 2

The user reviews the spec in chat and approves it. When they say so explicitly, in one change:

- `- Status: approved`
- a first `## Revision log` line: `- YYYY-MM-DD: Gate 2 approved by <name> in chat.`

Silence, or "carry on", is not approval. Approval needs `## Open questions` to hold only `None stated.`. If any question is unanswered, ask for the answer first. Fold each answer into the section it changes (an AC, a Non-goal, the Format), note it under `## Design decisions` as decided by the user, and remove the question. Until approval, no plan is written.

## Done when

- `$work/spec.md` exists with every template heading and `Status: draft` (or `approved` after Gate 2).
- Every AC appears under Verification.
- `## Critique` gives a disposition for every finding, and every open question is in the spec and in the report.
