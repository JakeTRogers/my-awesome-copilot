---
name: validate-pr-review-comments
description: 'Inspect and evaluate GitHub PR review comments, respond with a clear technical position, and implement only valid feedback. Use when asked to review PR comments, respond to reviewers, or make changes based on PR feedback.'
---

# Validate PR Review Comments

## Purpose

Use the GitHub CLI `gh` to inspect review comments on the current pull request, evaluate each comment independently, respond with a clear technical position, and only implement changes for review comments that are valid.

When changes are made, analyze the current branch’s commits to determine whether the edits should be selectively fixup’d into specific existing commits or left as a single final fixup commit for the user to autosquash.

## Core Principles

- Do not blindly implement reviewer suggestions.
- Only implement feedback that is technically valid and appropriate for the codebase.
- Always reply to each relevant review comment with a clear position.
- Keep replies concise, professional, and technically grounded.
- Do not make unrelated changes.
- Do not submit reviews, resolve threads, dismiss comments, push branches, or create commits unless explicitly instructed.
- Never attempt to create commits by default.

## Important Commit Constraint

The assistant must not create commits unless explicitly instructed.

The user requires signed commits, and the private signing key is stored on a YubiKey that requires physical touch. Because the assistant cannot perform that physical confirmation, it must not assume it can commit changes.

Instead, when code changes are made:

1. Modify the working tree as needed.
2. Analyze which existing commit or commits the changes logically belong to.
3. Provide suggested `git commit --fixup ...` commands for the user to run manually.
4. If there is only one commit on the current branch, state that the user can create a single fixup commit at the end and autosquash it.
5. If multiple commits exist, use commit history and `git blame` to determine whether changes should be selectively fixup’d into specific commits.

## Expected Workflow

### 1. Identify the Current PR

Use `gh` and `git` to identify the current pull request and branch context.

Useful commands include:

```bash
gh pr view --json number,title,author,headRefName,baseRefName,reviewDecision,url
```

```bash
git branch --show-current
```

```bash
git status --short
```

If the repository, branch, PR, GitHub authentication, or permissions are unavailable, stop and ask the user for clarification.

### 2. Collect Review Comments

Use `gh` to inspect current PR review comments.

Useful commands include:

```bash
gh pr view --comments
```

```bash
gh pr view --json number,title,reviews,comments
```

```bash
gh api repos/:owner/:repo/pulls/<PR_NUMBER>/comments
```

```bash
gh api repos/:owner/:repo/issues/<PR_NUMBER>/comments
```

```bash
gh api repos/:owner/:repo/pulls/<PR_NUMBER>/reviews
```

Prefer unresolved or currently relevant comments where possible. If the API does not clearly distinguish resolved from unresolved comments, process all relevant review comments and avoid duplicate responses.

### 3. Evaluate Each Comment

For each review comment:

1. Identify:
- Reviewer
- File path
- Line or diff hunk
- The reviewer’s concern
- The surrounding code
- Any related tests or call sites

2. Inspect the actual code before deciding whether the comment is valid.

3. Decide whether the comment is:
- Valid
- Invalid
- Partially valid
- Already addressed
- Obsolete due to later changes

Do not assume the reviewer is correct.

### 4. Decision Criteria

Treat a review comment as valid if it identifies one or more of the following:

- A correctness bug
- A security issue
- A reliability or edge-case problem
- A maintainability improvement aligned with the project
- A readability improvement with clear benefit
- A performance issue with practical impact
- A violation of established project conventions
- A missing or insufficient test where the risk justifies one

Treat a review comment as invalid or unnecessary if:

- It conflicts with existing project conventions
- It introduces unnecessary complexity
- It is based on a misunderstanding of the code
- It changes behavior outside the PR’s scope
- It is purely stylistic and unsupported by project standards
- It would reduce correctness, clarity, performance, or maintainability

### 5. Reply to Each Review Comment

Add a concise reply to each relevant review comment stating the position.

When agreeing:

```text
Agreed. This is valid because [brief reason]. I’ll address it with [brief approach].
```

When disagreeing:

```text
I don’t think this change is needed because [brief technical reason]. The current approach is appropriate given [context].
```

When partially agreeing:

```text
Partially agreed. The concern about [specific part] is valid, but I don’t think [other part] is necessary because [reason]. I’ll address this by [approach].
```

When already addressed:

```text
This appears to already be addressed by [brief explanation or commit/change reference]. No further change needed.
```

After implementation, if a second reply is appropriate:

```text
Addressed by [brief description of change]. Validated with [command].
```

Prefer replying directly to the specific review comment or thread when possible.

Do not resolve or dismiss review threads unless explicitly instructed.

### 6. Implement Only Valid Feedback

Only make code changes when the review comment is valid or partially valid.

When implementing:

- Use independent judgment.
- Do not blindly apply the reviewer’s proposed solution.
- Prefer the simplest correct solution.
- Preserve existing architecture, APIs, naming, formatting, and conventions.
- Keep the change scoped to the review comment.
- Do not make drive-by refactors.
- Add or update tests when the change affects behavior or risk warrants coverage.

If multiple valid comments affect the same code, coordinate the implementation so the final result is coherent and avoids redundant edits.

### 7. Analyze Commit Ownership for Changes

If any files are changed, inspect the current branch’s commit structure before summarizing.

First determine the base branch and commits on the current branch.

Useful commands:

```bash
gh pr view --json baseRefName,headRefName
```

```bash
git merge-base HEAD origin/<base-branch>
```

```bash
git log --oneline <merge-base>..HEAD
```

If the base branch is not available locally, fetch it:

```bash
git fetch origin <base-branch>
```

Then inspect changed files:

```bash
git status --short
```

```bash
git diff
```

For each changed file or relevant changed hunk, use `git blame` and commit history to determine which existing commit introduced or last materially touched the affected code.

Useful commands:

```bash
git blame -L <start>,<end> -- <file>
```

```bash
git log --oneline -- <file>
```

```bash
git log --oneline -L <start>,<end>:<file>
```

```bash
git show <commit> -- <file>
```

Determine whether the changes should be:

1. Fixup’d into one specific existing commit
2. Split into multiple fixup commits targeting different existing commits
3. Left as one final fixup commit because there is only one commit on the branch
4. Left as a new standalone commit suggestion if the change is genuinely independent of existing commits

Do not actually create these commits unless explicitly instructed.

### 8. Commit Recommendation Rules

If there is only one commit on the branch:

- Do not create a commit.
- State that the user can create one fixup commit at the end and autosquash it.
- Provide the suggested command:

```bash
git commit --fixup <commit-sha>
```

If there are multiple commits on the branch:

- Use `git blame`, `git log`, and the changed hunks to identify the correct target commit or commits.
- If all changes belong to one existing commit, recommend one fixup command.
- If changes belong to multiple commits, explain how to split the working tree changes and provide suggested fixup commands for each target commit.
- If the changes cannot be confidently mapped to a specific commit, say so and recommend that the user review the diff manually before creating fixup commits.

Suggested commands may include:

```bash
git add <paths>
git commit --fixup <target-commit-sha>
```

For partial staging:

```bash
git add -p <file>
git commit --fixup <target-commit-sha>
```

For autosquash:

```bash
git rebase -i --autosquash <merge-base>
```

Because commits require user-controlled signing through a YubiKey, the assistant must leave these commands for the user to run manually.

### 9. Validate the Changes

After making changes, run relevant validation where practical.

Depending on the project, this may include:

```bash
npm test
npm run lint
npm run typecheck
npm run format:check
```

```bash
pnpm test
pnpm lint
pnpm typecheck
```

```bash
yarn test
yarn lint
```

```bash
go test ./...
```

```bash
cargo test
cargo clippy
cargo fmt --check
```

```bash
pytest
ruff check .
mypy .
```

Use the repository’s documented commands when available.

If full validation is impractical, run the most targeted relevant checks and clearly state what was not run.

### 10. Final Summary

At the end, provide a concise summary using this format:

```markdown
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
- Branch commits reviewed: [yes/no]
- Commit structure: [single commit / multiple commits / unable to determine]
- Blame/history reviewed: [yes/no, with brief detail]
- Recommended fixup strategy:
- [Suggested target commit or commits]
- [Suggested commands for the user to run manually]

Example:

```bash
git add <paths>
git commit --fixup <commit-sha>
git rebase -i --autosquash <merge-base>
```

### Notes
- [Any risks, assumptions, unresolved comments, or follow-up items]
```

## Failure and Clarification Conditions

Stop and ask the user for clarification if:

- There is no current Git repository.
- No current PR can be identified.
- `gh` is unavailable or unauthenticated.
- The PR comments cannot be retrieved.
- The working tree already has unrelated changes and it is unclear whether they are user-owned.
- The base branch or merge base cannot be determined.
- A requested action would require committing, signing, pushing, resolving threads, dismissing reviews, or submitting a PR review without explicit user approval.

## Non-Goals

Do not:

- Commit changes by default.
- Push changes.
- Resolve or dismiss review threads.
- Submit a final PR review.
- Make unrelated refactors.
- Implement invalid feedback just to satisfy a reviewer.
- Change public behavior outside the PR’s scope unless required for correctness.
