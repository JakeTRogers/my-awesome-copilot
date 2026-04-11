---
name: conventional-commit
description: 'Generate Conventional Commit messages from provided staged Git context. Use when asked to write a commit message, summarize staged changes, classify a change as feat/fix/refactor/docs/test/build/ci/style/perf.'
allowed-tools: [read, shell(git diff, git log, git status)]
---

# Generate Conventional Commit messages from staged Git context

Use this skill to draft a Conventional Commit message from provided Git context such as `git status`, staged file lists, and `git diff --cached` output.

## When to Use This Skill

- User asks for a commit message
- User asks for a Conventional Commit
- User provides staged Git context
- User wants help choosing the correct Conventional Commit type

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

Use an optional scope when it improves clarity. You can use `git log <file>` to see past scopes used for a file or directory.

## Drafting Rules

- Base the message on the provided staged-change context.
- If the staged diff is ambiguous, read repository files only as needed to clarify intent.
- Prefer a single commit message that captures the primary reason for the staged changes.
- Use imperative mood.
- Keep the subject under 72 characters.
- Add a body only when the subject alone is not enough.
  - If you add a body, use markdown lists if it helps readability.
- Add a footer only for breaking changes or issue references.
- If there are no staged changes in the provided context, output exactly `NO_STAGED_CHANGES`.
- DO NOT output any explanation, reasoning, or commentary. Only the final commit message

## Output Rules

Output only the complete commit message as plain text:
- no Markdown fences
- no labels
- no commentary

Format:

`<type>(<scope>): <description>`

or

`<type>: <description>`

Optional body and footer may follow standard Git commit message formatting.

## Gotchas

- **Always** base the message on the current staged-change context, not unstaged or hypothetical changes.
- **Prefer one primary purpose** when multiple files are staged. Do not list every file in the subject line.
- **Do not** treat formatting or dependency updates as `feat` or `fix` unless the context clearly shows that.
- **Use `build`** for dependency or build tooling changes.
- **Use `ci`** for workflow and pipeline changes.
- **Use `refactor`** for restructuring that does not change behavior.
- **Only use `!` or `BREAKING CHANGE:`** when the change is actually breaking.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| The diff looks too large or mixed | Focus on the primary purpose of the staged changes and use the body for important secondary details. |
| The type is unclear from the diff alone | Read only the relevant repository files needed to clarify the intent. |
| The output is too verbose | Return only the final commit message, with no explanation. |

## References

- Conventional Commits: https://www.conventionalcommits.org/
