---
name: conventional-commit
description: 'Generate Conventional Commit messages by inspecting current staged Git changes or using provided staged Git context. Use when asked to write a commit message, summarize staged changes, or classify a change as feat/fix/refactor/docs/test/build/ci/style/perf.'
---

# Generate Conventional Commit messages from staged Git context

Use this skill to draft a Conventional Commit message from the current staged changes or provided Git context such as `git status`, staged file lists, and `git diff --cached` output.

## When to Use This Skill

- User asks for a commit message
- User asks for a Conventional Commit
- User provides staged Git context
- User wants help choosing the correct Conventional Commit type

## Repository Workflow

When repository access is available, run these commands in order at the start of every invocation, even if the skill already ran in the same session:

```bash
PAGER=cat git status
PAGER=cat git diff --cached
PAGER=cat git log --author="$(git config user.name)" --pretty=format:'%s' --no-merges -30
```

- Treat `git diff --cached` as the source of truth. Use status and history only as context; never include unstaged or untracked changes in the message.
- If the staged diff is empty, stop and ask whether the user wants to stage files or inspect a different diff target. Do not draft a commit message.
- If repository access is unavailable, use provided staged-change context. Ask for that context when none is available.
- Use recent authored subjects to match the user's established style and scope conventions without overriding this skill's rules.

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

- Base the message on the current staged diff or provided staged-change context.
- If the staged diff is ambiguous, read repository files only as needed to clarify intent.
- Prefer a single commit message that captures the primary reason for the staged changes.
- Use imperative mood.
- Keep the subject under 72 characters.
- Add a body only when the subject alone is not enough.
  - If you add a body, use markdown lists if it helps readability.
- Add a footer only for breaking changes or issue references.
- When a message can be drafted, do not output explanation, reasoning, or commentary. Output only the final commit message.
- Beware of pagination in git and GitHub cli, set `PAGER=cat` and `GH_PAGER=cat`.

## Output Rules

Output only the complete commit message inside one fenced Markdown code block:
- use a `text` language identifier so the response has a copy button
- include the entire commit message in the block, including any body or footer
- optional explanation or commentary may appear outside the block, but it must not be part of the suggested commit message

Format:

```text
<type>(<scope>): <description>
```

or

```text
<type>: <description>
```

Optional body(prefer markdown lists for readability) and footer may follow standard Git commit message formatting.

## Gotchas

- **Always** base the message on the current staged-change context, not unstaged or hypothetical changes.
- **Prefer one primary purpose** when multiple files are staged. Do not list every file in the subject line.
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
