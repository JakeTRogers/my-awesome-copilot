---
name: conventional-commit
description: 'Draft Conventional Commit messages from local Git changes: one message for staged changes, a revised message for the last commit, or staging groups with a message each for unstaged work; inside a .worktrees/ worktree it also stages and commits them unsigned, outside one it hands the user the commands. Use when asked for a commit message, to reword or amend the last commit message, how to split or stage changes into commits, or which Conventional Commit type (feat/fix/refactor/docs/test/build/ci/style/perf) fits. Not for fixups to existing branch commits (use fixup-plan) or pull request text (use pull-request).'
---

# Conventional Commit

## Purpose

Draft Conventional Commit messages and get each one committed: outside a worktree the user reviews and edits it in their own `git commit` editor; in a worktree you commit it yourself (see Constraints). Use this skill for staged changes, for rewording or amending the last commit, and for splitting uncommitted work into commits. Use fixup-plan for changes that belong in existing branch commits and pull-request for PR text.

## Constraints

- First check whether you are in a worktree: one whose top level sits under `<repo root>/.worktrees/`. Any other checkout, including a linked worktree elsewhere, counts as outside. This prints which:

  ```bash
  case "$(git rev-parse --show-toplevel)/" in
    "$(dirname "$(git rev-parse --path-format=absolute --git-common-dir)")/.worktrees/"*) echo worktree ;;
    *) echo outside ;;
  esac
  ```

- **Outside a worktree**, run only read-only commands: `git status`, `git diff`, `git log`, `git show`, `git config` lookups, and `cz check`. Changing the index, working tree, or history (`git add`, `git commit`, `git stash`, `git restore`, `git reset`) is left to the user, who signs commits with a hardware key outside a worktree and edits every message before committing.
- **In a worktree**, also stage and commit with the drafted message, unsigned: `git -c commit.gpgsign=false commit -F <message-file>` (add `--amend` in Amend mode). The user re-signs the branch after it lands. Two exceptions stay read-only, as outside a worktree:
  - Raw mode, or a prompt that supplies the context or says not to run commands (step 1): the caller commits with the returned message.
  - Amend mode when HEAD is also on another branch. A new worktree branch starts at the start branch's tip, and amending that commit stops the branch from fast-forwarding onto it. Amend only when `git --no-pager branch --contains HEAD` lists the current branch alone; otherwise output the message and leave the rewording to the user.
- Leave tagging to the user: create no tags in either case.
- Run every git command as `git --no-pager <command>`. `PAGER=cat` is ignored when `core.pager` is set, and a pager blocks agent terminals.

## Workflow

### 1. Gather context

Run these at the start of every invocation, even if they ran earlier in the session:

```bash
git --no-pager status --untracked-files=all
git --no-pager diff --cached --stat
git --no-pager log --no-merges --format=%s -30 --author="$(git config user.name)" --invert-grep --grep='^bump:'
```

Then run `git --no-pager diff --cached`. If `--stat` shows more than 500 changed lines, instead diff only the paths that inform the message with `git --no-pager diff --cached -- <path>`, and skip lockfiles, generated files, and vendored code.

- If the prompt already provides git status and diff output (for example, from a wrapper script) or says not to run commands, use the provided context and run no commands in any step.
- In a repository without commits, `git log` fails; continue without history.
- To read a staged file, use `git --no-pager show :<path>`; the working-tree copy may contain unstaged edits. Describe only changes that appear in the diff.

### 2. Select the mode

Use the first row that matches:

| # | Condition | Mode |
|---|-----------|------|
| 1 | `git status` lists unmerged paths | Stop: resolve conflicts first |
| 2 | `git status` reports a merge, cherry-pick, or revert in progress (a rebase in progress does not count) | Stop: keep the message git prepared |
| 3 | The user asks to amend or reword the last commit | Amend |
| 4 | The user asks to split or group all changes, including staged ones | Group (all changes) |
| 5 | The staged diff is non-empty | Staged |
| 6 | There are unstaged or untracked changes | Group (unstaged and untracked) |
| 7 | Otherwise | Stop: nothing to commit |

An empty staged diff means nothing is staged; it says nothing about whether the repository has commits.

- **Staged:** the staged diff is authoritative; ignore unstaged and untracked changes. If the staged changes match a group you proposed earlier in this conversation, reuse that group's message verbatim.
- **Amend:** the message must describe everything the amended commit will contain. Also run:

  ```bash
  git --no-pager log -1 --format=%B
  git --no-pager diff --cached HEAD~1
  ```

  If HEAD has no parent, use `git --no-pager show HEAD` plus the staged diff instead. Treat the current message as a draft: keep what is accurate and correct the rest.
- **Group:** also run `git --no-pager diff --stat` and `git --no-pager diff` (for all changes, `git --no-pager diff HEAD --stat` and `git --no-pager diff HEAD`), read untracked files whose contents affect grouping or the message, then follow step 3.

### 3. Group changes

Group mode only:

1. Put changes in the same commit when reverting one without the other would leave the repository broken or inconsistent, such as a feature with its tests and docs. Otherwise, separate them.
2. Include untracked files; `git diff` does not show them.
3. When one file holds changes for more than one group, stage it with `git add -p -- <shell-quoted-file>` and name the hunks that belong to each group by their `@@` header or enclosing function.
4. Order groups so each commit works on its own, prerequisites first.
5. If the grouping is genuinely ambiguous, ask one focused question before producing commands. If you cannot ask, use the fewest commits that satisfy rule 1 and state that assumption in one line.

### 4. Classify

**Type.** Decide in order:

1. If the change modifies code beyond comments (source, scripts, or AI instruction files such as prompts, skills, agent definitions, and `*.instructions.md`), classify by effect, not by file extension:
   - `feat`: adds a capability
   - `fix`: corrects wrong behavior
   - `perf`: same behavior, faster or cheaper
   - `style`: formatting or whitespace only
   - `refactor`: same behavior, restructured

   Tests, docs, and dependency changes that accompany the code do not change its type.
2. Otherwise, use the category of the primary change:
   - `test`: tests only
   - `docs`: human-facing documentation only, such as READMEs, guides, and code comments
   - `build`: build system, packaging, dependency manifests and lockfiles, dev containers, and repository tooling config such as `.gitignore`, `.editorconfig`, linter and formatter config, and release config (`.cz.yaml`)
   - `ci`: CI workflows, git and pre-commit hooks, Dependabot or Renovate config, and version bumps inside any of these

Never use `chore`. Mark a change as breaking only when existing users or consumers must change something: add `!` before the colon and a `BREAKING CHANGE: <what breaks and how to migrate>` footer.

**Scope.** Optional; never more than one:

1. The scope used most often for the primary path in `git --no-pager log -20 --format=%s -- <path>`, or in the provided recent subjects when commands are not allowed. Use its most frequent spelling.
2. Otherwise, the component or directory name that best identifies what changed, such as `parser` or `skills`.
3. Omit the scope when the change spans several areas.

Match the wording and scope names of the recent subjects, but follow these rules where they disagree.

### 5. Write the message

- **Header:** `type(scope): description`, with `!` before the colon for breaking changes.
  - Keep the whole line at 72 characters or fewer, including type and scope; aim for 60.
  - Use the imperative mood, a lowercase first word unless it is an identifier or proper noun, and no trailing period.
  - State the primary purpose; do not list files.
- **Body:** add one only when the reason is not obvious from the header or there are two or more notable changes. Separate it from the header with a blank line and use `- ` bullets, at most five, wrapped at 72 characters. Explain what changed and why, not file by file. Use no markdown headings, and start no line with `#`.
- **Footer:** only `BREAKING CHANGE:` and issue references such as `Refs: #123` or `Closes #123`. Take issue numbers only from the user, the branch name, or the diff; never invent them. Never add `Co-authored-by`, `Signed-off-by`, or tool attribution trailers; GPG signing is not a sign-off.
- **Validate:** when commands are allowed, shell-quote the generated header as one argument and run `cz check -m <shell-quoted-header>`, revising the header until it passes. Skip this if `cz` is not installed.

### 6. Output

Use raw mode when the invoking prompt asks for it, for example a script that passes the output to `git commit -F`. Otherwise, use chat mode.

#### Chat mode

- **Staged and Amend:** output exactly one `text` fenced block containing the complete message: header, body, and footer. Output nothing else, except at most one line after the block, starting with `Note:`, when:
  - the staged changes look like more than one logical change,
  - a path has both staged and unstaged changes, or
  - in Amend mode, `git status` says the branch is up to date with or behind its upstream, so amending requires a force push.
- **Group:** for each group, in order:
  1. One line: `Commit <n> of <total>: <why these changes belong together>`, naming any partial-file hunks.
  2. One `bash` fenced block with the staging commands: `git add -- <shell-quoted-path>...` for whole files and `git add -p -- <shell-quoted-file>` for selected hunks. Shell-quote every path as one argument using POSIX single-quote escaping, including paths without whitespace. When grouping all changes and the index is not empty, start Commit 1's block with `git restore --staged -- :/`.
  3. One `text` fenced block with the complete message.

  Outside a worktree, the user runs the commits, so the output carries staging commands and messages only. In a worktree, run each group's staging commands and commit it unsigned before staging the next group.
- **Stop:** one sentence stating the reason.

#### Raw mode

- Output the complete message as plain text: no fences, labels, notes, or commentary.
- Only Staged and Amend modes produce a message. In any other mode, or when no message can be drafted, output exactly one line: `NO_COMMIT_MESSAGE: <short reason>`.

#### Examples

Staged mode, chat mode:

````markdown
```text
fix(parser): handle empty quoted values

- treat "" as an empty string instead of a missing value
- add regression tests for empty and whitespace-only values
```
````

Group mode, chat mode:

````markdown
Commit 1 of 2: the empty-value fix and its regression test; in `src/parser.py`, stage only the hunk in `parse_quoted()`.

```bash
git add -- 'tests/test_parser.py'
git add -p -- 'src/parser.py'
```

```text
fix(parser): handle empty quoted values
```

Commit 2 of 2: the remaining `src/parser.py` hunk renames an internal helper and is independent of the fix.

```bash
git add -- 'src/parser.py'
```

```text
refactor(parser): rename tokenize helper to split_tokens
```
````

## References

- [Conventional Commits](https://www.conventionalcommits.org/)
