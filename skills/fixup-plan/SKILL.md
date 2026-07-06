---
name: fixup-plan
description: 'Map uncommitted working-tree changes to the existing branch commits they belong to, then produce git commit --fixup and autosquash rebase commands for the user to run. Use when asked which commit a change belongs to, to plan fixup commits, to organize changes into fixups, to prepare an autosquash rebase, or after implementing review feedback on a branch with multiple commits.'
---

# Fixup Plan

## Purpose

Given uncommitted changes on a feature branch, determine which existing on-branch commit each change logically belongs to, and output the `git commit --fixup` commands that fold them in cleanly. This mirrors an interactive fixup helper: blame the changed lines, match them to on-branch commits, group files by target, and finish with an autosquash rebase.

## Constraints

- **Never create commits, tags, or rebases yourself.** The user requires GPG-signed commits via a YubiKey that needs physical touch, so all `git commit` and `git rebase` commands must be left for the user to run manually.
- Only analyze and stage-plan; do not `git add` unless explicitly asked.
- Never target a `fixup!`, `squash!`, or `amend!` commit — always target the original commit.

## Workflow

### 1. Establish the branch context

Find the merge-base with the base branch and list the on-branch commits, excluding autosquash commits:

```bash
base_branch=$(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')
merge_base=$(git merge-base HEAD "origin/${base_branch}")
git log --format='%h %s' --grep='^fixup! ' --grep='^squash! ' --grep='^amend! ' --invert-grep "${merge_base}..HEAD"
```

If there are no on-branch commits, stop: there is nothing to fixup, so recommend a normal new commit instead.

### 2. Map each change to a target commit

For each changed file (`git status --short`, `git diff HEAD`). Diff against `HEAD`, not the index: plain `git diff` misses anything already staged, and its old-side line numbers would not match a blame of `HEAD`.

1. Prefer blame of the changed lines: for each modified hunk, run `git blame HEAD -L <start>,<end> -- <file>` using the old-side line numbers from `git diff -U0 HEAD`, and keep only commits that appear in the `${merge_base}..HEAD` list. The commit that last materially touched the changed lines is the default target. If blame lands on a pre-branch commit instead, the change edits base-branch code — check the file's on-branch history before declaring it new work.
2. For added-only hunks (no old-side lines), blame the immediately surrounding context lines instead — the commit that owns the enclosing code is usually the right target.
3. Fall back to file history when blame is still inconclusive (e.g. moved code): `git log --format='%h %s' "${merge_base}..HEAD" -- <file>`. A single on-branch commit touching the file is an automatic target.
4. New/untracked files have no fixup target by default; assign them to the on-branch commit that introduced the feature they extend, or recommend a standalone commit if they are genuinely independent.
5. Renamed files: analyze history under the old path, but stage both old and new paths together in the same fixup.

### 3. Classify the outcome

- **Single commit on branch** — one fixup at the end is enough; no per-file analysis needed.
- **Multiple commits, clean mapping** — group files (or hunks, via `git add -p`) by target commit, one fixup per target.
- **Ambiguous mapping** — say so explicitly and recommend the user review the diff before choosing; do not guess silently.
- **Genuinely new work** — recommend a standalone conventional commit instead of a fixup.

### 4. Output the plan

Present a grouped summary, then the exact commands. Example:

```text
a1b2c3d feat(parser): add array support
  - src/parser.go        (blame: lines 40-55 introduced by a1b2c3d)
  - src/parser_test.go   (only on-branch commit touching file)
e4f5a6b fix(cli): handle empty input
  - cmd/root.go          (blame: lines 12-14)
```

```bash
git add src/parser.go src/parser_test.go
git commit --fixup a1b2c3d

git add cmd/root.go
git commit --fixup e4f5a6b

git rebase -i --autosquash <merge-base>
```

If the branch is already pushed, remind the user that the autosquash rebase rewrites history, so the final push needs `git push --force-with-lease`.

Use `git add -p <file>` in the plan when a single file's hunks belong to different target commits.

## Gotchas

- Blame the **pre-image** of changed lines: pass `HEAD` explicitly (`git blame HEAD -L <start>,<end> -- <file>`) and use the old-side line numbers from `git diff -U0 HEAD`. Blaming the working tree (no revision) attributes modified lines to "Not Committed Yet".
- Blame can land on an existing `fixup!` commit already on the branch (they are hidden from the target list but still exist in history). Redirect to the commit it amends — match the subject after the `fixup! ` prefix — and target that instead.
- `git symbolic-ref refs/remotes/origin/HEAD` fails when `origin/HEAD` is unset (common in older clones) or there is no remote. Fix with `git remote set-head origin --auto`, or fall back to the local `main`/`master` branch.
- `git log -L <start>,<end>:<file>` is a useful cross-check for line-range history but can be slow on large files — prefer targeted `git blame -L`.
- If the base branch is not available locally, `git fetch origin <base-branch>` first.
- If the working tree mixes fixup-bound changes with unrelated user-owned edits, list the unrelated files separately and leave them out of the plan.
