---
name: tollpoint-intent
description: Captures an idea, issue, or incident as a local .tollpoint/<slug>/intent.md, adds the intent-critic's questions to a medium or high risk intent, and records Gate 1 (accept or reject) when the user decides in chat. Use when the user has a new idea, says 'new intent', brings an issue into the workflow, says critique the intent, or accepts or rejects an intent.
---

# tollpoint-intent

Stage 1 of Tollpoint. Reads an idea, a GitHub issue, or an incident note and writes `.tollpoint/<slug>/intent.md`. Gate 1 (accept or reject) belongs to the person in the chat, never to this skill. Nothing here is committed.

## Work folder

Tollpoint keeps its files at the main checkout, so a worktree sees the same ones:

```bash
common="$(git rev-parse --path-format=absolute --git-common-dir)"
root="$(dirname "$common")"
work="$root/.tollpoint/<slug>"
```

On every run, make sure `.tollpoint/` is ignored locally, without touching the repo's `.gitignore`:

```bash
mkdir -p "$common/info"
grep -qxF '.tollpoint/' "$common/info/exclude" 2>/dev/null || echo '.tollpoint/' >> "$common/info/exclude"
```

## Inputs

Accept any of:

- Free text: `/tollpoint-intent let people tag ideas`.
- An issue number: read it with `gh issue view 12 --json number,title,body`. Only read it; this skill never edits issues or labels.
- A file path: an incident note or a pasted request.

## Rules

- **Never invent requirements.** Every statement in the intent must trace to the input or to an answer the originator gave in this session. If a section would need a guess, write the question under `## Open questions` instead.
- An intent says what and why, not how. No acceptance criteria (that is the spec), no designs, no file lists (that is the plan).
- The slug is the ID: two to four kebab-case words from the title. If `.tollpoint/<slug>/` already exists, update that intent rather than creating a second one.
- Do not set `Status: accepted` or `rejected` unless the user gives that decision explicitly in this session (see "Gate 1").
- An intent whose status is `accepted` or `rejected` is never rewritten. Append a dated `## Revision (YYYY-MM-DD)` section instead.

## Steps

1. **Pick the slug** and check `ls "$root/.tollpoint/"` for an existing intent on the same subject.
2. **Read the input** in full.
3. **Find the gaps.** For each template section, note whether the input answers it. Gaps that matter most: who feels the problem, what observably changes, risk, and what is out of scope.
4. **Ask, once.** Ask at most five clarifying questions, all in one message, most important first. Offer a default for each where the input suggests one. Do not ask about anything the input already answers. Zero questions is fine for a complete brief.
5. **Write the intent** to `$work/intent.md` from the template below:
   - `Author`: the originator (the person who asked, or the issue author), not the agent.
   - `Date`: today, `YYYY-MM-DD`.
   - `Source`: `idea`, `issue #N`, `incident`, or `retro` (an intent written after the work was done).
   - `Risk`: use the rubric below; if unsure, pick the higher level and say why under Open questions.
   - `Status: draft`.
   - Remove the template's guidance lines; keep every heading. A section with nothing to say gets `None stated.`, never filler.
6. **Critique** if `Risk:` is `medium` or `high` (see "Critique"). A `low` intent skips it unless the user asks.
7. **Report** to the user: the file path, the risk and why, every open question, the critic's questions listed apart from the intent's own, the `Intent critic:` line, and the next step: the user accepts or rejects it.

## Template

```markdown
# Intent: <short title>

- Slug: <slug>
- Author: <name>
- Date: YYYY-MM-DD
- Source: idea | issue #N | incident | retro
- Risk: low | medium | high
- Status: draft | accepted | rejected

## Problem
What cannot be done today, and who feels it.

## Proposed outcome
What better looks like, observably.

## Affected users and systems

## Constraints

## Out of scope

## Open questions
Questions that must be resolved before the spec.
```

## Risk rubric

- `low`: documentation, or a change contained in one module; easy to revert.
- `medium`: changes shared tooling, a workflow other work depends on, or user-visible behavior.
- `high`: touches production, deploys, secrets, data, security controls, or repository enforcement (branch rules, environments, required checks).

## Critique

Dispatch the read-only `intent-critic` agent with only the intent's path and the instruction "Critique this intent". Do not summarize the intent or say what you think of it.

- In Claude Code, dispatch it as `tollpoint:intent-critic`; in Cursor, as `intent-critic`.
- If that fails, dispatch a generic read-only subagent with: "Read `~/.cursor/plugins/local/tollpoint/agents/intent-critic.md` and follow it exactly on `<path>`. Do not edit files or run commands." Record this as `intent-critic (<harness>, stand-in)`.
- One pass per draft. If `## Open questions` already holds `Intent critic, ` questions and an `Intent critic:` line from an earlier run, replace them. If any of them has an answer under it, stop and ask before replacing, so no answer is lost.
- For each row whose Question is not `none`, append to `## Open questions`, continuing its numbering (replace `None stated.` if that is all it holds):
  `N. Intent critic, <topic>: <question> Options: <options>. Evidence: <evidence>. No default; answer at Gate 1.`
- Then append one unnumbered line as the section's last line:
  `Intent critic: intent-critic (<harness>[, stand-in]), YYYY-MM-DD, <N> question(s); none on <topics whose Question is none>.`
  Leave out the `; none on ...` part when every row is a question. If no dispatch works, the line is `Intent critic: not run (<reason>).`

On request ("critique the intent", "run the intent critic on <slug>"), run this whatever the risk:

- `draft`: as above.
- `accepted` with no `spec.md` beside it: append a `## Revision (YYYY-MM-DD)` section that starts "Intent critique, run on request; Gate 1 stands.", then the questions numbered from 1 and ending "No default; answer before the spec." instead, then the record line. `Status:` stays `accepted`.
- `rejected`, or a `spec.md` beside it: write nothing and say why. A spec's framing is argued by `spec-critic`.

## Gate 1

The user reviews the intent in chat and says accept or reject. When they do, set `Status: accepted` or `Status: rejected`, and for a rejection add the reason they gave as a `## Revision (YYYY-MM-DD)` section. Silence, "carry on", or "use the defaults" is not a decision. Until then the intent stays `draft` and no spec is written.

Before recording an accept, every intent critic question needs an answer from the user. A critic question has no default. Record each answer inline under its question; an answer that changes the Proposed outcome, Constraints, or Out of scope also gets a dated `## Revision (YYYY-MM-DD)` section.

## Done when

- `$work/intent.md` exists with every template heading, and `.tollpoint/` is in `.git/info/exclude`.
- Every open question is in the intent and in the report.
- A `medium` or `high` intent's `## Open questions` ends with an `Intent critic:` line.
