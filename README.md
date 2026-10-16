# Tollpoint

Tollpoint adds design-stage gates to agent work in Claude Code and Cursor. You capture an intent, harden it into a spec, and turn the spec into a plan. A read-only critic argues against each draft, and you approve each stage in chat. Once you've built the change, Tollpoint reviews the diff against the spec's acceptance criteria. Its working files live in `.tollpoint/` and Tollpoint does not commit them.

## How it works

```text
idea / issue / incident
  → tollpoint-intent → intent.md → Gate 1 (accept / reject)
  → tollpoint-spec   → spec.md   → Gate 2 (approve)
  → tollpoint-plan   → plan.md   → Gate 3 (approve)
  → build (your way)
  → tollpoint-review → AC checklist in the PR body
```

Each gate is decided by the person in chat, never by a skill or an agent:

- **Gate 1:** you accept or reject the intent.
- **Gate 2:** you approve the spec.
- **Gate 3:** you approve the plan.

Each of the three design stages writes nothing until the previous gate has passed. For example, running `tollpoint-spec` on an intent that is still `draft` writes no spec and tells you Gate 1 has not passed. After Gate 3, build however you like, then run `tollpoint-review`.

## Skills

| Skill | Writes | What it does |
|---|---|---|
| [`tollpoint-intent`](skills/tollpoint-intent/SKILL.md) | `intent.md` | Captures an idea, a GitHub issue, or an incident note as an intent: what and why, not how. Adds the intent-critic's questions to a medium or high risk intent, or at any risk level on request. Records Gate 1 when you accept or reject it in chat. |
| [`tollpoint-spec`](skills/tollpoint-spec/SKILL.md) | `spec.md` | Turns an accepted intent into a spec with numbered, testable acceptance criteria. Hardens it with the spec-critic and records the critique in the spec. Records Gate 2 when you approve it in chat. |
| [`tollpoint-plan`](skills/tollpoint-plan/SKILL.md) | `plan.md` | Turns an approved spec into a plan: the files that change and ordered tasks, each tied to acceptance criteria and a verification, plus risks, proof, and rollback. Records Gate 3 when you approve it in chat. |
| [`tollpoint-review`](skills/tollpoint-review/SKILL.md) | AC checklist in the PR body | Reviews the branch diff against the spec and plan with the code-reviewer. Writes an acceptance criteria checklist (met, unmet, or later) into the PR body, or prints it if there is no PR. |

## Agents

| Agent | Dispatched by | What it does |
|---|---|---|
| [`intent-critic`](agents/intent-critic.md) | `tollpoint-intent` | Argues against a draft intent's framing before Gate 1. Returns one question each on the cheapest route to its outcome, whether the work can ever be complete, and what it costs every later session, each with evidence from the repo. |
| [`spec-critic`](agents/spec-critic.md) | `tollpoint-spec` | Argues against a draft spec before you approve it. Returns weaknesses, a failure mode, alternative designs, untestable acceptance criteria, missing non-goals, and unbacked context claims, each with a severity. |
| [`code-reviewer`](agents/code-reviewer.md) | `tollpoint-review` | Reviews a branch diff for bugs, security problems, and compliance with the spec and plan. Returns one-line findings and a per-AC table (met, unmet, or later). Never approves or merges. |

All three agents are read-only. In Claude Code, `tools: Read, Grep, Glob` enforces this. In Cursor, `readonly: true` is advisory only, so the agents' own instructions are what keep them read-only.

## The .tollpoint/ folder

Each piece of work gets a folder at `.tollpoint/<slug>/` in the main checkout, so a worktree sees the same files. The folder holds `intent.md`, `spec.md` and `plan.md`.

Tollpoint does not commit these files. Every `tollpoint-intent` run adds `.tollpoint/` to `.git/info/exclude`, so the folder stays out of `git status` without changing the repo's `.gitignore`. The exclude only hides the folder; it does not stop a deliberate `git add -f`.

`tollpoint-review` copies each acceptance criterion's text into the PR body. That text is as public as the PR.

## Install

### Claude Code

```text
/plugin marketplace add lstod/Tollpoint
/plugin install tollpoint@tollpoint
```

### Cursor

Choose one route:

1. **Local plugin folder.** Clone the repo, then run "Developer: Reload Window":

   ```bash
   git clone https://github.com/lstod/Tollpoint ~/.cursor/plugins/local/tollpoint
   ```

2. **Through Claude Code.** Install the plugin in Claude Code as above. Then turn on "Include Third-Party Plugins, Skills, and Other Configs" in Cursor Settings and run "Developer: Reload Window".

The skills call each agent by name. If that fails, they fall back to reading the agent's file from `~/.cursor/plugins/local/tollpoint/agents/`, and that path exists only with the local plugin folder install.

## Usage

Start with an intent:

- Claude Code: `/tollpoint:tollpoint-intent let people tag ideas`
- Cursor: `/tollpoint-intent let people tag ideas`

Once you accept the intent, move on to `tollpoint-spec`, then `tollpoint-plan`, then build, then `tollpoint-review`.

## Related

Tollpoint is a cut-down version of PRdex, a private workflow kit. It keeps PRdex's design stage and code review and drops the rest.

## License

MIT. See [LICENSE](LICENSE).
