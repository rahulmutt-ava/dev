---
name: responding-to-pr-review
description: >-
  Use when responding to, addressing, or drafting replies to pull-request
  review comments or reviewer feedback — including "address the review",
  "reply to the comments on my PR", "the reviewer left feedback", or "draft
  responses to <reviewer>". Fetches the PR's unresolved review comments, skips
  any already drafted in ./.claude/pr-review/COMMENTS.md, triages each one,
  runs the brainstorm → spec → plan → implement cycle for comments that need a
  code change (subagent-driven by default), and drafts a reply per comment.
  Invoke even when the user just says "go through the review comments" without
  naming this workflow.
---

# Responding to PR review comments

Reviewers leave comments; each one needs either a **reply** (questions, acks,
praise) or a **code change plus a reply** (requested fixes, bugs, design
concerns). This skill drives that end to end: it gathers the open comments,
ignores anything you've already drafted a reply for, decides what each comment
actually needs, makes the change properly when one is required, and leaves a
draft reply you can review before posting.

The output is **draft replies only** — written to `./.claude/pr-review/COMMENTS.md`.
This skill never posts to GitHub or resolves threads. The human reviews
`COMMENTS.md` and posts manually.

## Process

Create a todo per step so nothing is dropped.

### 1. Gather the open review comments

Try `gh` first; fall back to asking the user to paste.

- Identify the PR for the current branch: `gh pr view --json number,url,reviews,title`.
- Pull review comments (the inline, line-anchored ones reviewers leave on the
  diff):
  ```sh
  gh api "repos/{owner}/{repo}/pulls/<number>/comments" --paginate
  ```
  and the top-level review bodies from `gh pr view --json reviews`.
- Prefer **unresolved** threads. The REST comments endpoint doesn't expose
  resolution state directly; if you need it, query review threads via
  `gh api graphql` (`reviewThreads { isResolved }`). If that's awkward, gather
  all comments and rely on the dedup in step 2 to skip handled ones.

**Fallback:** if there's no PR, no `gh`, or auth fails, ask the user to paste
the review comments (or a link), and parse them from the conversation.

Record for each comment: a stable identifier (the GitHub comment `id` when
available), the file + line, the reviewer, and the verbatim text. You'll quote
the verbatim text in the reply.

### 2. Skip comments already addressed

Read `./.claude/pr-review/COMMENTS.md` if it exists. Each comment section
carries a marker — match incoming comments against it and **drop any that are
already drafted**:

```markdown
## Comment 3 — `hooks_test.go`: test through the VM?
<!-- comment-id: 1234567890 -->
```

Match on `comment-id` when you have it; otherwise match on the quoted text /
file+line. Only the remaining (un-drafted) comments proceed. If every comment
is already covered, say so and stop.

### 3. Triage each remaining comment

Decide what the comment needs before doing any work:

- **Reply only** — a question, a clarification request, an acknowledgment, a
  "why did you…", or praise. No code change. Draft a direct, honest answer.
- **Code change + reply** — a requested fix, a bug, a missing test, a design
  concern, a "this should…". These go through the full cycle in step 4.

Before treating a code-change comment as correct, apply
**`superpowers:receiving-code-review`**: verify the feedback technically rather
than agreeing performatively. If the reviewer is mistaken or the suggestion is
questionable, the right response is a reply that explains the reasoning with
evidence — not a code change. Genuine disagreement, surfaced respectfully with
proof, is a valid outcome.

### 4. The cycle for code-change comments

Run each comment that needs code through brainstorm → spec → plan → implement.
Default to handling them one at a time so each reply reflects exactly what
changed; batch only tightly-related comments that touch the same code.

1. **Brainstorm** — invoke **`superpowers:brainstorming`** to explore the
   problem and candidate solutions with the user before committing to an
   approach. This is required before creative/implementation work; don't skip
   to coding.
2. **Spec + plan** — once the approach is agreed, invoke
   **`superpowers:writing-plans`** to turn it into a written spec and
   step-by-step implementation plan. For a multi-comment review this naturally
   produces one plan covering the agreed changes.
3. **Implement — subagent-driven by default.** Execute the plan with
   **`superpowers:subagent-driven-development`** (dispatch implementation tasks
   to subagents in the current session). Use **`superpowers:executing-plans`**
   instead when the user wants a separate session with review checkpoints, or
   when the plan is large enough to warrant its own branch/worktree (see
   **`superpowers:using-git-worktrees`**).
   - During implementation, follow **`superpowers:test-driven-development`** —
     write the failing test first, then the code.
   - If the comment is about a bug or unexpected behavior, route through
     **`superpowers:systematic-debugging`** before proposing a fix.
4. **Verify** — before claiming the comment is addressed, apply
   **`superpowers:verification-before-completion`**: run the relevant
   tests/lint and confirm the output. For this repo that means the matching
   `./scripts/run_task.sh` targets (e.g. `test-unit`, `lint`, or `lint-saevm`
   for `vms/saevm/`). Optionally run
   **`superpowers:requesting-code-review`** on the change before drafting the
   reply.

### 5. Draft the reply

Write one reply per comment into `./.claude/pr-review/COMMENTS.md`, following
the format below. The reply should be honest and specific:

- State what changed and why, referencing concrete files/symbols.
- Include the key code snippet when it clarifies the fix.
- **Surface caveats and limits honestly** — what the fix does *not* cover,
  follow-ups worth filing, or where you disagree with the reviewer and why.
  Don't overclaim. Reviewers trust replies that admit the edges.

### 6. Hand off

Tell the user the drafts are in `./.claude/pr-review/COMMENTS.md`, summarize
which comments got code changes vs. reply-only, and note any you disagreed with
or left as follow-ups. Posting is theirs to do.

## COMMENTS.md format

Create `./.claude/pr-review/COMMENTS.md` if absent. Use this structure so
re-runs can dedup and the human can read top-to-bottom:

```markdown
# Draft replies to <reviewer>'s review on PR #<number>

## Comment <N> — `<file>` <short title>
<!-- comment-id: <id-or-"pasted"> -->

> <verbatim quote of the reviewer's comment>

<your reply: what changed and why, with code snippets and honest caveats>
```

Append new comment sections; never rewrite or drop ones already present unless
the user asks you to revise a specific reply.

## Notes

- **Draft-only is a hard boundary.** Don't run `gh pr comment`, reply to
  threads, or resolve anything. Surfacing the drafts for human review is the
  whole point.
- Process skills come first: brainstorming and systematic-debugging decide
  *how* to approach a comment before any implementation skill runs.
- If the user explicitly says "just reply, no code" for a comment, honor that —
  user instructions outrank this workflow.
