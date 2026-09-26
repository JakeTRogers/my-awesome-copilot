---
name: conventional-commit
description: 'Generate Conventional Commit messages from staged or unstaged Git changes. Use when asked for a commit message, commit grouping, staging advice, a change summary, or help classifying a commit as feat/fix/refactor/docs/test/build/ci/style/perf.'
---

# Generate Conventional Commit messages from Git changes

Use this skill to draft Conventional Commit messages from current Git changes or provided Git context. When staged changes exist, use only the staged diff. When there are no staged changes, help the user decide whether the unstaged work belongs in one commit or several related commits, then provide the staging commands and complete message for each proposed commit.

## When to Use This Skill

- User asks for a commit message
- User asks for a Conventional Commit
- User provides staged or unstaged Git context
- User asks how to group changes into commits or what to stage
- User wants help choosing the correct Conventional Commit type


## Repository Workflow

When repository access is available, run these commands in order at the start of every invocation, even if the skill already ran in the same session:

```bash
PAGER=cat git status
PAGER=cat git diff --cached
PAGER=cat git log --author="$(git config user.name)" --pretty=format:'%s' --no-merges -30
```

- Use the index state to select exactly one workflow:
  - **Staged workflow:** If `git diff --cached` is non-empty, treat it as authoritative. Base the message only on staged changes, even when unstaged or untracked changes also exist.
  - **Unstaged workflow:** If `git diff --cached` is empty, inspect tracked changes with `PAGER=cat git diff`. Use `git status --short` and `git ls-files --others --exclude-standard` to identify untracked paths, then inspect untracked files when their contents affect grouping or the message.
  - **Clean workflow:** If the index and working tree are both clean, do not invent a message. Ask the user to provide or stage changes.
- “No staged changes” means that `git diff --cached` is empty. It does not refer to whether the repository has commits in its history.
- If the repository has no commits, the `git log` command may fail; continue with no history and use the available status, diff, and file contents. Do not treat missing history as a clean working tree.
- If repository access is unavailable, use the provided Git context. Determine whether it describes staged changes, unstaged changes, or neither; ask for the missing context when necessary.
- Use recent authored subjects to match the user's established style and scope conventions without overriding this skill's rules.

### Unstaged Change Grouping

When the unstaged workflow applies:

1. Decide whether the changes express one coherent purpose or several unrelated purposes. Group files together when they implement, document, test, or configure the same logical change. Separate changes when they can be reviewed, reverted, or released independently.
2. Include untracked files in the analysis when they are part of the working-tree changes. Do not silently omit them just because `git diff` does not display them.
3. When different logical changes occur in the same file, do not suggest staging the whole file. Use `git add -p -- <file>` and explain which hunks belong to the current group.
4. If the grouping is genuinely ambiguous, explain the uncertainty and ask one focused clarification question before producing staging commands. Do not present a speculative grouping as certain.
5. Do not run `git add`, `git commit`, or any other command that changes the user's working tree. Present the commands for the user to run.

## Commit Type Rules

Choose one of these types:

- `feat` — introduces a new feature
- `fix` — patches a bug
- `docs` — documentation only
- `style` — formatting or non-semantic code style changes
- `refactor` — code restructuring without bug fix or new feature
- `perf` — performance improvement
- `test` — add or correct tests
- `build` — build system or dependency changes
- `ci` — CI configuration or workflow changes

Never use `chore` — pick the most specific type from the list above instead.

Use an optional scope when it improves clarity. Never use multiple scopes in a single commit; if a change spans areas, pick the primary one or omit the scope. You can use `git log <file>` to see past scopes used for a file or directory.

## Drafting Rules

- For the staged workflow, base the message on the current staged diff or provided staged-change context.
- For the unstaged workflow, base each message on the files or hunks assigned to that group. Include untracked-file contents when relevant.
- If the relevant diff is ambiguous, read repository files only as needed to clarify intent. If the intended grouping remains ambiguous, ask a focused question rather than guessing.
- Prefer a single commit message that captures the primary reason for the staged changes.
- Use imperative mood.
- Keep the subject under 72 characters.
- Add a body only when the subject alone is not enough.
  - If you add a body, use markdown lists if it helps readability.
- Add a footer only for breaking changes or issue references.
- In the staged workflow, output only the final commit message as specified below. In the unstaged workflow, a brief grouping rationale is allowed when it helps explain why changes were combined or separated.
- Beware of pagination in git and GitHub cli, set `PAGER=cat` and `GH_PAGER=cat`.

## Output Rules

Choose the output contract that matches the repository state. The proposed message is always the complete commit message, including its subject and any body or footer; it is not only the body.

### Staged Workflow

Output only the complete commit message inside one fenced Markdown code block:

- Use a `text` language identifier so the response has a copy button.
- Include the entire message in the block, including any body or footer.
- Do not include staging or commit commands, labels, or explanation.

Format:

```text
<type>(<scope>): <description>
```

or

```text
<type>: <description>
```

Optional body(prefer markdown lists for readability) and footer may follow standard Git commit message formatting.

### Unstaged Workflow

If all unstaged changes form one logical commit, output one group. If they belong in multiple commits, output one group for each logical commit. For every group:

1. Optionally state a brief rationale and identify the paths or hunks in the group.
2. Output one `bash` fenced block containing the staging command or commands for that group:
  - Use `git add -- <paths>` for complete files.
  - Use `git add -p -- <file>` when only selected hunks from a file belong in the group.
3. Immediately follow it with one `text` fenced block containing the complete proposed commit message for that group.

Repeat this command-and-message pair for every group. Do not combine the staging command and commit message in one block, and do not include a `git commit` command.

Example with unrelated files:

````markdown
Group 1: The parser change and its test belong together.

```bash
git add -- src/parser.py tests/test_parser.py
```

```text
feat(parser): support quoted values
```

Group 2: The documentation update is independent.

```bash
git add -- docs/usage.md
```

```text
docs: clarify quoted value syntax
```
````

Example with unrelated hunks in one file:

````markdown
Group 1: Stage only the parser hunks from `src/parser.py`.

```bash
git add -p -- src/parser.py
```

```text
fix(parser): handle empty quoted values
```
````

## Gotchas

- **Always** choose the workflow from the current index state: staged changes take precedence; an empty index invokes unstaged grouping.
- **Prefer one primary purpose** when multiple files are staged. Do not list every file in the subject line.
- Do not infer “no commits exist” from an empty staged diff. The relevant condition is whether the index contains changes.
- Do not omit untracked files from unstaged grouping decisions.
- Do not stage an entire file when its hunks belong to different logical commits; use `git add -p -- <file>`.
- **Do not** treat formatting or dependency updates as `feat` or `fix` unless the context clearly shows that.
- **Use `build`** for dev container or build tooling changes.
- **Use `ci`** for workflow, hook, and pipeline changes.
- **Use `refactor`** for restructuring that does not change behavior.
- **Only use `!` or `BREAKING CHANGE:`** when the change is actually breaking.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| The diff looks too large or mixed | Focus on the primary purpose of the staged changes and use the body for important secondary details. |
| The type is unclear from the diff alone | Read only the relevant repository files needed to clarify the intent. |
| The output is too verbose | Return only the final commit message, with no explanation. |

## References

- [Conventional Commits](https://www.conventionalcommits.org/)
