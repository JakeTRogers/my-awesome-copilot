---
name: implement-spec
description: 'Implement a spec and its tickets on one local integration branch: implementer subagents build tickets in parallel worktrees under .worktrees/, and each finished ticket lands by rebase and fast-forward. Use when asked to implement a spec or its tickets end to end.'
disable-model-invocation: true
metadata:
  upstream: 'https://github.com/mattpocock/skills/tree/v1.3.2/skills/engineering/implement-spec'
---

# Implement Spec

Adapted from the `implement-spec` skill in [mattpocock/skills](https://github.com/mattpocock/skills) at `v1.3.2`. This version keeps history linear and leaves every signature and push to the maintainer: implementers rebase, the orchestrator fast-forwards, and nothing leaves the machine during the run.

## Ground rules

- The tickets are a **task graph** with blocking relationships, not a list of steps. At any moment a **frontier** of unblocked tickets is ready to grab.
- The repo's `AGENTS.md` says when agents may commit, where worktrees live, and how work lands. Follow it. The issue tracker is named there too; if it is missing, ask the maintainer which tracker holds the tickets.
- You, the orchestrator, are the only one who creates worktrees, moves the integration branch, and removes worktrees. Create each worktree with `git worktree add` yourself; leave the agent runtime's own worktree isolation switched off.
- History stays linear: work lands by rebase and fast-forward only. Pushing, opening the PR, and signing all belong to the maintainer after the run.
- When a ticket's work conflicts with these rules (it needs a push, a merge commit, a tag, or a PR to proceed), stop that ticket and ask the maintainer how to proceed.

Start every subagent with a fresh context window. Keep messages to and from subagents sparse: pass **context pointers** (the spec, the ticket, research notes, commit SHAs, the worktree path) instead of restating what they point to. Run implementer subagents in the background for maximum concurrency.

## Steps

1. **Map the graph.** Read the spec and every ticket. Confirm the `tdd`, `conventional-commit`, and `code-review` skills are installed; if any is missing, ask the maintainer which skill or process to use in its place before starting. Done when you can name each ticket's blockers and the current frontier.

2. **Explore (optional).** When tickets need codebase or external research, dispatch one exploration subagent that saves its markdown notes in a directory outside the repo, readable by every later subagent. Implementers then point at those notes instead of re-exploring.

3. **Create the integration branch** from the current branch, locally: `git branch <integration> HEAD`. It is never pushed during the run.

4. **Start an implementer for each frontier ticket.** For ticket `<n>`, create its worktree on a new `ticket/<n>` branch from the integration tip:

   ```bash
   git worktree add -b ticket/<n> "<repo root>/.worktrees/ticket-<n>" <integration>
   ```

   Then dispatch an implementer subagent with the worktree's absolute path, the ticket pointer, and the integration branch name. Its instructions:
   - Work only in that worktree. First confirm `git merge-base --is-ancestor <integration> HEAD`; if it fails, rebase onto the integration tip with `git -c commit.gpgsign=false rebase <integration>`, and if that rebase stops on a conflict, run `git rebase --abort` and report back instead of building.
   - Invoke the `tdd` skill to build the ticket.
   - Commit with the `conventional-commit` skill, which in this worktree drafts the message and commits it unsigned itself; do not commit again afterwards.
   - Before reporting done, rebase onto the integration tip with `git -c commit.gpgsign=false rebase <integration>`.
   - Report the commit SHAs and anything unresolved. Leave merging, pushing, tagging, PRs, and issue comments to the orchestrator and maintainer.

5. **Land each finished ticket** as soon as its implementer reports:
   1. Fast-forward the integration branch to `ticket/<n>`: `git merge --ff-only ticket/<n>` where the integration branch is checked out, or `git fetch . ticket/<n>:<integration>` where it is not.
   2. If the fast-forward is refused because the integration branch moved, send the same implementer back to rebase onto the new tip and report again, then retry. Never resolve it with a merge.
   3. Once landed, remove the worktree and its branch: `git worktree remove "<repo root>/.worktrees/ticket-<n>"`, then `git branch -d ticket/<n>` (use `-D` only after `git merge-base --is-ancestor ticket/<n> <integration>` confirms it landed).

   Done when the integration tip contains the ticket's commits and its worktree and branch are gone.

6. **Advance the frontier.** When a landing unblocks tickets, start implementers for them as in step 4. Repeat steps 4 to 6 until every ticket has landed. Track blockers against the graph from step 1, not the tracker: a blocker is satisfied once it is closed or has landed on the integration branch, because landed tickets stay open until the maintainer's PR closes them.

7. **Review.** Invoke the `code-review` skill on the integration branch against the branch it was cut from. Fix every issue raised in a single implementer subagent, in its own worktree, landed the same way as step 5.

8. **Report.** Confirm `git worktree list` shows none of this run's worktrees and no `ticket/*` branches from this run remain. Then give the maintainer:
   - the integration branch name and its commits ahead of the start branch;
   - the tickets it implements, so the PR body can close them;
   - the finishing steps, in order: re-sign the branch with `gresign`, push it, then open the PR (for example with `/pull-request`).
