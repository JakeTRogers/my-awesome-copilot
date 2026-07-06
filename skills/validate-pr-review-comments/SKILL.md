---
name: validate-pr-review-comments
description: 'Inspect and evaluate GitHub PR review comments, respond with a clear technical position, and implement only valid feedback. Use when asked to review PR comments, respond to reviewers, address reviewer feedback, or make changes based on PR feedback.'
---

# Validate PR Review Comments

## Purpose

Triage the review comments on the current pull request using the GitHub CLI `gh`: evaluate each comment on its technical merits, implement only the feedback that holds up, reply to each thread with a clear position, and map any resulting edits to fixup commits for the user to run.

## Prerequisites

- Authenticated GitHub CLI (`gh auth status`) with access to the repository.
- The PR branch checked out locally, with no unrelated uncommitted changes.

## Core Principles and Constraints

- Evaluate before implementing. Reviewers act in good faith but rarely have full context — a suggestion can be wrong, already handled, or out of scope. Blindly applying every suggestion degrades the code, and a well-reasoned "no" is often the more useful reply.
- Reply to every triaged comment with a concise, professional, technically grounded position — whether or not it results in a change. Replies are immediately visible to reviewers and cannot be quietly retracted, so draft the full set locally and post only after the user approves it.
- Keep changes scoped to the review feedback; no unrelated changes or drive-by refactors.
- **Never create commits.** The user's commits are GPG-signed with a YubiKey that requires physical touch, so no commit can be completed here. Modify the working tree only and hand the user suggested `git commit --fixup` commands to run manually.
- Do not push branches, resolve or dismiss review threads, or submit a final PR review unless explicitly instructed.

## Workflow

### 1. Identify the Current PR

```bash
gh pr view --json number,title,author,headRefName,baseRefName,reviewDecision,url
git branch --show-current
git status --short
```

If the repository, branch, PR, or `gh` authentication cannot be established, stop (see Failure and Clarification Conditions).

### 2. Collect Unresolved Review Comments

Resolved-vs-unresolved state is only exposed by the GraphQL API, so fetch the review threads with their resolution state and REST comment IDs. Use `--paginate` so PRs with more than 100 threads are not silently truncated — it follows `pageInfo` and emits one JSON document per page (add `--slurp` to combine them into a single array):

```bash
pr_number=$(gh pr view --json number -q .number)
gh api graphql --paginate -F owner='{owner}' -F repo='{repo}' -F pr="$pr_number" -f query='
  query($owner: String!, $repo: String!, $pr: Int!, $endCursor: String) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        reviewThreads(first: 100, after: $endCursor) {
          pageInfo { hasNextPage endCursor }
          nodes {
            isResolved
            isOutdated
            path
            line
            comments(first: 50) {
              totalCount
              nodes { databaseId author { login } body diffHunk }
            }
          }
        }
      }
    }
  }'
```

Process threads where `isResolved` is `false`. If every thread is already resolved and there is no unaddressed PR-level feedback, report that there is nothing to triage and stop. Keep each thread's first `databaseId` — it is the REST comment ID needed to reply to that thread in step 6. If a thread's `totalCount` exceeds 50, say so — its remaining comments were not fetched. Treat `isOutdated: true` as a hint that later commits may have already addressed the concern; verify against the current code before classifying the thread as obsolete.

For PR-level (non-inline) discussion and review summaries, supplement with:

```bash
pr_number=$(gh pr view --json number -q .number)
gh pr view --comments
gh api "repos/{owner}/{repo}/pulls/${pr_number}/reviews"
```

### 3. Evaluate Each Comment

For each unresolved comment:

1. Identify the reviewer, file path, line or diff hunk, and the reviewer's actual concern.
2. Inspect the current code, related tests, and call sites before taking a position — the diff hunk stored with the comment may no longer match the branch.
3. Classify the comment as: valid, partially valid, invalid, already addressed, or obsolete.

Treat a comment as valid when it identifies:

- A correctness bug
- A security issue
- A reliability or edge-case problem
- A maintainability improvement aligned with the project
- A readability improvement with clear benefit
- A performance issue with practical impact
- A violation of established project conventions
- A missing or insufficient test where the risk justifies one

Treat a comment as invalid or unnecessary when it:

- Conflicts with existing project conventions
- Introduces unnecessary complexity
- Is based on a misunderstanding of the code
- Changes behavior outside the PR's scope
- Is purely stylistic and unsupported by project standards
- Would reduce correctness, clarity, performance, or maintainability

Judge each comment on its merits alone — neither the reviewer's confidence nor the ease of the change makes it valid.

### 4. Implement Only Valid Feedback

Make code changes only for valid or partially valid comments. When implementing:

- Use independent judgment; the reviewer's proposed fix is one option, not the required one.
- Prefer the simplest correct solution.
- Preserve existing architecture, APIs, naming, formatting, and conventions.
- Keep the change scoped to the review comment.
- Add or update tests when the change affects behavior or the risk warrants coverage.
- Apply reviewer `suggestion` blocks by editing the working tree, adapting them where the surrounding code has moved on — never through GitHub's apply-suggestion API, which creates a commit on the branch.

When multiple valid comments touch the same code, implement them together so the final result is coherent rather than a series of overlapping edits.

### 5. Validate the Changes

Run the repository's documented test/lint/typecheck commands (from its README, Makefile, CI config, or AGENTS.md) against the affected code. If full validation is impractical, run the most targeted relevant checks and state clearly what was not run.

### 6. Reply to Each Review Comment

Draft a reply for every triaged comment, present the full set to the user, and wait for approval before posting any of them — the user may reword or drop individual replies. Replying after implementation and validation means each reply states a final position instead of a promise that later work might contradict.

Reply directly on the review thread, not with a top-level PR comment. `gh pr comment` posts to the PR conversation, which detaches the reply from the thread — use the replies endpoint with the thread's REST comment ID (`databaseId` from step 2):

```bash
pr_number=$(gh pr view --json number -q .number)
gh api "repos/{owner}/{repo}/pulls/${pr_number}/comments/<COMMENT_ID>/replies" -f body='...'
```

The templates below are shapes, not boilerplate. Fill them with the specific technical reasoning for that comment — a reply that could be pasted under any comment tells the reviewer nothing.

When agreeing (change implemented):

```text
Agreed — [brief reason]. Addressed by [brief description of change]. Validated with [command].
```

When disagreeing:

```text
I don't think this change is needed because [brief technical reason]. The current approach is appropriate given [context].
```

When partially agreeing:

```text
Partially agreed. The concern about [specific part] is valid and addressed by [brief description of change], but I don't think [other part] is necessary because [reason].
```

When already addressed:

```text
This appears to already be addressed by [brief explanation or commit/change reference]. No further change needed.
```

### 7. Plan Fixup Commits for the Changes

If any files changed, use the **fixup-plan** skill when it is installed; otherwise apply its approach:

- Find the merge-base with the PR's base branch and the on-branch commits (excluding `fixup!`/`squash!`/`amend!` commits).
- Use `git blame` on the changed lines and per-file `git log` to pick each change's target commit.
- With only one commit on the branch, recommend a single `git commit --fixup <sha>` at the end.
- With changes mapping to multiple commits, group files (or `git add -p` hunks) per target and recommend one fixup per target, ending with `git rebase -i --autosquash <merge-base>`.
- If a change cannot be confidently mapped, say so and recommend the user review the diff manually.

All fixup and rebase commands are suggestions for the user to run — never execute them (see Core Principles and Constraints).

### 8. Final Summary

Close with this template, omitting any section with no entries:

````markdown
## PR Review Comment Triage Summary

### Implemented

- [Comment/topic]: [what changed and why]

### Not Implemented

- [Comment/topic]: [why no change was made]

### Partially Implemented

- [Comment/topic]: [what was addressed and what was not]

### Already Addressed or Obsolete

- [Comment/topic]: [why no further action was needed]

### Files Changed

- `path/to/file`

### Validation

- Ran: `[commands]`
- Result: [passed/failed/not run]

### Commit / Fixup Recommendation

- Commit structure: [single commit / multiple commits / unable to determine]
- Recommended fixup strategy: [target commit(s) and rationale]
- Suggested commands:

```bash
git add <paths>
git commit --fixup <commit-sha>
git rebase -i --autosquash <merge-base>
```

### Notes

- [Any risks, assumptions, unresolved comments, or follow-up items]
````

## Failure and Clarification Conditions

Stop and ask the user for clarification if:

- There is no current Git repository, or no current PR can be identified.
- `gh` is unavailable or unauthenticated, or the PR comments cannot be retrieved.
- The working tree already has unrelated changes and it is unclear whether they are user-owned.
- The base branch or merge base cannot be determined.
- A requested action would require committing, signing, pushing, resolving threads, dismissing reviews, submitting a PR review, or posting replies the user has not approved.
