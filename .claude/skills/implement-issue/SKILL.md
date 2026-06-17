---
name: implement-issue
description: >-
  Use when asked to implement, work on, take on, or "do" a GitHub issue —
  including "implement issue #123", "implement #123", "work on this issue",
  "pick up <issue-url>", or "let's tackle issue 456". Fetches the issue,
  researches the relevant code in the repo, runs superpowers:brainstorming to
  sketch design choices and field open questions to the user, then drives the
  agreed approach to completion through the rest of the superpowers:* workflow
  (writing-plans, worktrees, TDD, subagent/executing-plans, verification, code
  review, finishing the branch). Invoke even when the user just names an issue
  number without saying "implement".
---

# Implementing a GitHub issue

A GitHub issue is a problem statement, not a solution. This skill drives one
end to end: fetch the issue, understand the relevant code, **brainstorm design
choices and surface open questions to the user before committing to an
approach**, then implement the agreed solution properly with tests, review, and
a clean integration.

**The brainstorming gate is not optional.** Even when the issue looks
mechanical or the design "seems obvious," you run
`superpowers:brainstorming` and get explicit user sign-off on the approach
before writing implementation code. Hard-to-reverse decisions (data formats,
APIs, schemas, public interfaces) especially must be agreed first.

## Process

Create a todo per step so nothing is dropped. Process skills come first —
brainstorming and systematic-debugging decide *how* to approach the work before
any implementation skill runs.

### 1. Fetch the issue

Get the full issue — never work from the title alone.

```sh
gh issue view <number> --json number,title,body,labels,url,comments,author
```

Read the body, acceptance criteria, labels, linked ACPs/specs/PRs, and the
comment thread for design discussion. If a URL was given instead of a number,
pass it to `gh issue view`. 

**Fallback:** if there's no `gh`, auth fails, or it's a different forge, ask the
user to paste the issue text or link, and parse it from the conversation.

Record the concrete requirement and any stated acceptance criteria — you'll
check the finished work against them in step 7.

### 2. Research the relevant context in the repo

Map the code the issue touches *before* designing anything. Dispatch
**`Explore`** subagents (or `general-purpose` agents) in parallel for breadth —
you want the conclusions, not file dumps. Establish:

- The packages/files the change lives in and the surrounding conventions
  (naming, error handling, test style) to match.
- Existing types, interfaces, and patterns the solution should reuse or mirror.
- Any project docs, invariants, or stricter lint passes that apply to the area
  (check `CLAUDE.md` and any local `docs/`).
- Current branch state: `git log master..HEAD --oneline` and
  `git diff master --stat` — scaffolding may already exist. Check for an
  associated PR with `gh pr list --head $(git branch --show-current)`.

If the issue describes a bug or unexpected behavior, route the investigation
through **`superpowers:systematic-debugging`** to find the root cause before
designing a fix.

### 3. Brainstorm design choices and field open questions — REQUIRED

Invoke **`superpowers:brainstorming`**. Use what you learned in steps 1–2 to:

- Sketch the candidate approaches with their trade-offs.
- **Surface every open question to the user** — ambiguities in the issue,
  competing designs, scope boundaries, edge cases, and any decision that's
  hard to reverse. Ask; don't assume.
- Converge on an agreed approach with explicit user sign-off.

Do not proceed to a plan or code until the approach is agreed. This is the gate
the issue's title can tempt you to skip — don't.

### 4. Write the spec and plan

Once the approach is agreed, invoke **`superpowers:writing-plans`** to turn it
into a written spec and a step-by-step implementation plan. The plan should map
directly to the issue's acceptance criteria so completion is verifiable.

### 5. Set up an isolated workspace

For anything beyond a trivial one-file change, invoke
**`superpowers:using-git-worktrees`** to create an isolated workspace branched
off `master` (or the appropriate base). This keeps the issue work separate from
the current tree. Skip only for tiny changes the user wants applied in place.

### 6. Implement

Pick the execution mode and follow it:

- **`superpowers:subagent-driven-development`** — default; dispatch the plan's
  independent tasks to subagents in the current session.
- **`superpowers:executing-plans`** — when the user wants a separate session
  with review checkpoints, or the plan is large.
- **`superpowers:dispatching-parallel-agents`** — when the plan has 2+ genuinely
  independent tasks with no shared state.

During implementation, follow **`superpowers:test-driven-development`**: write
the failing test first, then the code. If you hit a bug or unexpected behavior
mid-implementation, drop into **`superpowers:systematic-debugging`** rather than
patching blindly.

### 7. Verify against the issue

Before claiming anything is done, apply
**`superpowers:verification-before-completion`**: run the relevant tests, lint,
and build, and confirm the actual output. Check the result against the issue's
acceptance criteria from step 1 — evidence before assertions. For this repo
that means the matching `./scripts/run_task.sh` targets (e.g. `test-unit`,
`lint`, or `lint-saevm` for `vms/saevm/`), plus any `generate-*` task if you
changed mocks/proto/canoto and committing the regenerated output.

### 8. Code review

Invoke **`superpowers:requesting-code-review`** on the change. If the review
returns feedback, apply **`superpowers:receiving-code-review`** — verify each
point technically rather than agreeing performatively, and address real issues
with another TDD pass.

### 9. Finish the branch

Invoke **`superpowers:finishing-a-development-branch`** to choose how to
integrate the work — merge, open a PR (reference the issue, e.g. "Closes
#<number>"), or clean up. Let the user make the final integration call.

## Quick reference

| Step | Skill / tool |
|------|--------------|
| Fetch issue | `gh issue view` |
| Research context | `Explore` / `general-purpose` agents; `superpowers:systematic-debugging` for bugs |
| Design + open questions | **`superpowers:brainstorming`** (required gate) |
| Spec + plan | `superpowers:writing-plans` |
| Isolated workspace | `superpowers:using-git-worktrees` |
| Implement | `superpowers:subagent-driven-development` / `executing-plans` / `dispatching-parallel-agents` + `test-driven-development` |
| Verify | `superpowers:verification-before-completion` |
| Review | `superpowers:requesting-code-review` → `receiving-code-review` |
| Integrate | `superpowers:finishing-a-development-branch` |

## Common mistakes

- **Skipping brainstorming because the issue "looks simple."** The title hides
  the design decisions. Always run the brainstorming gate and get sign-off.
- **Assuming instead of asking.** Open questions go to the user in step 3, not
  resolved silently by guessing.
- **Coding before researching.** Map the existing code and conventions first so
  the solution fits the repo.
- **Claiming done without running anything.** Verification means real command
  output checked against the acceptance criteria.

## Notes

- User instructions outrank this workflow. If the user explicitly says "just
  write it, no brainstorming" or "apply it in place, no worktree," honor that.
- The brainstorming gate is the one step to defend hardest — it's the cheapest
  place to catch a wrong approach.
