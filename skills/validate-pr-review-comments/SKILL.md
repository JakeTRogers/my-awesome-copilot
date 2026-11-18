---
name: validate-pr-review-comments
description: 'Inspect and evaluate GitHub PR review comments, implement only valid feedback one comment at a time, identify the branch commit each fix belongs in so it can be committed as its own fixup, and reply with a clear technical position. Use when asked to review PR comments, respond to reviewers, address reviewer feedback, or make changes based on PR feedback.'
---

# Validate PR Review Comments

## Purpose

Triage the review comments on the current pull request using the GitHub CLI `gh`: evaluate each comment on its technical merits, then work through the valid ones one at a time — implement the fix, identify the branch commit it belongs in, and hand the user the fixup commands before starting the next comment. Finish by replying to each thread with a clear position.

Addressing comments one at a time keeps the working tree limited to a single fix whenever the user commits. Each fixup can stage whole files, so the user never has to split mixed changes across commits with `git add -p`.

## Prerequisites

- Authenticated GitHub CLI (`gh auth status`) with access to the repository.
- The PR branch checked out locally with a clean working tree (`git --no-pager status --short` prints nothing). Pre-existing changes would mix into the first fixup.

## Core Principles and Constraints

- Evaluate before implementing. Reviewers act in good faith but rarely have full context — a suggestion can be wrong, already handled, or out of scope. Blindly applying every suggestion degrades the code, and a well-reasoned "no" is often the more useful reply.
- Reply to every triaged comment with a concise, professional, technically grounded position — whether or not it results in a change. Replies are immediately visible to reviewers and cannot be quietly retracted, so draft the full set locally and post only after the user approves it.
- Keep changes scoped to the review feedback; no unrelated changes or drive-by refactors.
- **One comment, one fixup at a time.** Never start editing for the next comment while the previous fix is still uncommitted. Accumulated fixes for different target commits force the user into `git add -p`, which is what this workflow exists to avoid.
- **Never create commits.** The user's commits are GPG-signed with a YubiKey that requires physical touch, so no commit can be completed here. Modify the working tree only and hand the user suggested `git commit --fixup` commands to run manually.
- Do not push branches, resolve or dismiss review threads, or submit a final PR review unless explicitly instructed.
- Run every git command as `git --no-pager <command>` and every GitHub CLI command as `GH_PAGER=cat gh <command>`. `PAGER=cat` is ignored when `core.pager` is set, and a pager blocks agent terminals.

## Workflow

### 1. Identify the Current PR

```bash
GH_PAGER=cat gh pr view --json number,title,author,headRefName,baseRefName,reviewDecision,url
git --no-pager branch --show-current
git --no-pager status --short
```

If the repository, branch, PR, or `gh` authentication cannot be established, stop (see Failure and Clarification Conditions).

### 2. Collect Unresolved Review Comments

Resolved-vs-unresolved state is only exposed by the GraphQL API, so fetch the review threads with their resolution state and REST comment IDs. Use `--paginate` so PRs with more than 100 threads are not silently truncated — it follows `pageInfo` and emits one JSON document per page (add `--slurp` to combine them into a single array):

```bash
pr_number=$(GH_PAGER=cat gh pr view --json number -q .number)
GH_PAGER=cat gh api graphql --paginate -F owner='{owner}' -F repo='{repo}' -F pr="$pr_number" -f query='
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

Process threads where `isResolved` is `false`. If every thread is already resolved and there is no unaddressed PR-level feedback, report that there is nothing to triage and stop. Keep each thread's first `databaseId` — it is the REST comment ID needed to reply to that thread in step 7. If a thread's `totalCount` exceeds 50, say so — its remaining comments were not fetched. Treat `isOutdated: true` as a hint that later commits may have already addressed the concern; verify against the current code before classifying the thread as obsolete.

For PR-level (non-inline) discussion and review summaries, supplement with:

```bash
pr_number=$(GH_PAGER=cat gh pr view --json number -q .number)
GH_PAGER=cat gh pr view --comments
GH_PAGER=cat gh api "repos/{owner}/{repo}/pulls/${pr_number}/reviews"
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

Classify every comment before editing anything, then show the user the triage list: each comment's classification and the order you will address the valid ones in. Order the work by target commit, oldest first, when that is apparent. Comments that change the same lines form one work unit. Splitting them would create overlapping edits, and they almost always share a target commit.

### 4. Establish the Fixup Context

Find the merge-base and the on-branch commits once, so each fix can be mapped to a target:

```bash
base_branch=$(GH_PAGER=cat gh pr view --json baseRefName -q .baseRefName)
merge_base=$(git --no-pager merge-base HEAD "origin/${base_branch}")
git --no-pager log --format='%h %s' --grep='^fixup! ' --grep='^squash! ' --grep='^amend! ' --invert-grep "${merge_base}..HEAD"
```

### 5. Address Valid Comments One at a Time

Repeat this loop for each valid or partially valid work unit. Comments classified as invalid, already addressed, or obsolete need no loop — draft their replies (step 7) and move on.

#### 5a. Confirm a Clean Starting Point

Run `git --no-pager status --short` before every edit. If it is not empty, the previous fixup was not committed (or the user made other changes) — stop and ask rather than mixing changes.

#### 5b. Implement the Fix

- Use independent judgment; the reviewer's proposed fix is one option, not the required one.
- Prefer the simplest correct solution.
- Preserve existing architecture, APIs, naming, formatting, and conventions.
- Keep the change scoped to this comment — do not fold in fixes for other comments, even ones in the same file.
- Add or update tests when the change affects behavior or the risk warrants coverage.
- Apply reviewer `suggestion` blocks by editing the working tree, adapting them where the surrounding code has moved on — never through GitHub's apply-suggestion API, which creates a commit on the branch.

#### 5c. Validate the Fix

Run the most targeted relevant checks from the repository's documented test/lint/typecheck commands (README, Makefile, CI config, or AGENTS.md) so each fixup is known-good on its own. Record what ran for the reply and summary.

#### 5d. Identify the Target Commit

Use the **fixup-plan** skill when it is installed; otherwise apply its approach to this fix's diff:

- Blame the changed lines' pre-image (`git --no-pager blame HEAD -L <start>,<end> -- <file>`, using old-side line numbers from `git --no-pager diff -U0 HEAD`) and keep only commits in `${merge_base}..HEAD`. For added-only hunks, blame the surrounding context lines.
- Fall back to per-file history (`git --no-pager log --format='%h %s' "${merge_base}..HEAD" -- <file>`) when blame is inconclusive.
- With only one commit on the branch, that commit is the target.
- Blame may land on a `fixup!` commit the user created earlier in this loop. Target the original commit it amends, matching the subject after the `fixup! ` prefix.
- If the fix belongs to no existing commit (genuinely new work), recommend a standalone conventional commit instead of a fixup.
- If the target cannot be confidently determined, say so and ask the user to choose rather than guessing.

If the fix spans more than one target commit, do not hand over a mixed diff. Undo your edits back to the clean state confirmed in 5a, re-apply only the part that belongs to the first target, and finish this loop for it (5c-5f). Then repeat for the next target. Every handoff must stage cleanly with whole-file `git add`.

#### 5e. Draft the Reply

Draft this comment's reply now, while the reasoning is fresh, using the templates in step 7. Hold it for approval — do not post.

#### 5f. Hand Off and Wait

Present the comment (reviewer, file, concern), a short description of the change, the validation run, and the target commit with its rationale. Then give the exact commands. The working tree holds only this fix, so stage whole files:

```bash
git add <paths>
git commit --fixup <commit-sha>
```

For genuinely new work, give a standalone conventional commit message instead of `--fixup`.

Stop and wait for the user to confirm they committed, or to ask for revisions. Do not start the next comment until they do. Then check that `git --no-pager log -1 --format='%h %s'` shows the expected `fixup! <subject>` commit and the working tree is clean. Record the fixup's short SHA and target for the summary.

### 6. Validate the Combined Result

After the last fixup is committed, run the repository's full documented validation once against the branch head. Fixes validated one at a time can still interact. If full validation is impractical, state clearly what was not run. If something fails, address it as its own loop iteration (step 5).

### 7. Reply to Each Review Comment

Present the full set of drafted replies to the user and wait for approval before posting any of them — the user may reword or drop individual replies. Replies are posted after every fix is committed and validated, so each one states a final position instead of a promise that later work might contradict.

Reply directly on the review thread, not with a top-level PR comment. `gh pr comment` posts to the PR conversation, which detaches the reply from the thread — use the replies endpoint with the thread's REST comment ID (`databaseId` from step 2):

```bash
pr_number=$(GH_PAGER=cat gh pr view --json number -q .number)
GH_PAGER=cat gh api "repos/{owner}/{repo}/pulls/${pr_number}/comments/<COMMENT_ID>/replies" -f body='...'
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

### 8. Recommend the Autosquash Rebase

Once all fixups are committed, give the user the command to fold them into their targets — never run it:

```bash
git rebase -i --autosquash <merge-base>
```

If the branch is already pushed, remind the user that the rebase rewrites history, so the final push needs `git push --force-with-lease`.

### 9. Final Summary

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

### Fixups

| Comment/topic | Files | Fixup | Target commit |
| --- | --- | --- | --- |
| [topic] | `path/to/file` | `<fixup-sha>` | `<target-sha> <subject>` |

### Validation

- Per-fix: `[commands]` — [passed/failed]
- Combined: `[commands]` — [passed/failed/not run]

### Next Step

```bash
git rebase -i --autosquash <merge-base>
```

### Notes

- [Any risks, assumptions, unresolved comments, or follow-up items]
````

## Failure and Clarification Conditions

Stop and ask the user for clarification if:

- There is no current Git repository, or no current PR can be identified.
- `gh` is unavailable or unauthenticated, or the PR comments cannot be retrieved.
- The working tree is not clean at the start of the workflow or before the next comment's fix (step 5a).
- A fix's target commit cannot be confidently determined (step 5d).
- The base branch or merge base cannot be determined.
- A requested action would require committing, signing, pushing, resolving threads, dismissing reviews, submitting a PR review, or posting replies the user has not approved.
