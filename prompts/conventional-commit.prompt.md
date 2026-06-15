---
name: cc
description: 'Prompt and workflow for generating a Conventional Commit message from staged changes. Instructs Copilot to inspect git status and the staged diff, infer the primary change, and output a copy-friendly commit message.'
agent: agent
argument-hint: '[user provided context]'
model: GPT-5.4 mini (copilot)
tools: [execute/getTerminalOutput, execute/runInTerminal, read/readFile, search, vscode/askQuestions]
---

# Instructions

## Critical Rules

1. **ALWAYS run these commands at the start**. This is mandatory, not optional:
  - First: `git status`
  - Second: `git diff --cached`
  - Third: `git log --author="$(git config user.name)" --pretty=format:'%s' --no-merges -30`
2. **When this prompt is triggered again in the same session**, the user may have staged different or additional files. You MUST re-run all commands from step 1, even if you ran them before.
3. If multiple files are staged, generate a single commit message that identifies the primary reason for the changes.
4. If there are no staged changes, do not invent a commit message. Use `vscode/askQuestions` to ask the user whether they want to stage files first or inspect a different diff target.

## Conventional Commit Types

- fix: a bug fix that correlates with PATCH in Semantic Versioning
- feat: a new feature that correlates with MINOR in Semantic Versioning
- docs: documentation-only changes
- style: formatting or whitespace changes that do not affect behavior
- refactor: a code change that neither fixes a bug nor adds a feature
- perf: a code change that improves performance
- test: adding or correcting tests
- build: changes to the build system or external dependencies, for example `pip`, `docker`, or `npm`
- ci: changes to CI configuration files and scripts

BREAKING CHANGE: a commit with a `BREAKING CHANGE:` footer, or a `!` after the type or scope, introduces a breaking API change and correlates with MAJOR in Semantic Versioning. A breaking change can accompany any type.

## Workflow

Execute these steps in order. Do not skip any step.

1. Run `git status` in the terminal to review changed files.
2. Run `git diff --cached` in the terminal to inspect staged changes.
3. If `git diff --cached` is empty, stop and use `vscode/askQuestions` to ask the user how they want to proceed.
4. Analyze the output from both commands to understand what changed and why.
  - If you need more context, use `search` or `read/readFile` to inspect relevant files.
  - If the intent is ambiguous, use `vscode/askQuestions` to clarify before drafting the message.
5. Review the user's commit message style using `git log --author="$(git config user.name)" --pretty=format:'%s' --no-merges -30` to ensure your message is consistent with their commit history.
6. Construct the commit message with these components:
  - `type` required: one of `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`
  - `scope` optional: the component or area affected
  - `description` required: a short imperative summary
  - `body` optional: additional Markdown-formatted context only when the description is not enough
  - `footer` optional: breaking changes or issue references
7. Output the commit message in the exact format specified in the Final Output Format section.

## Example Commit Types

- `feat(parser): add ability to parse arrays`
- `fix(ui): correct button alignment`
- `docs: update README with usage instructions`
- `refactor: improve performance of data processing`
- `feat!: send email on registration`

## Validation

- `type`: Must be one of the allowed types.
- `scope`: Optional, but recommended for clarity.
- `description`: Required. Use the imperative mood, for example `add`, not `added`.
- `body`: Optional. Use it only when the description is not enough. Format it as a bulleted list.
- `footer`: Use it for breaking changes or issue references.

## Final Output Format

**CRITICAL**: Your response must contain ONLY the commit message blocks below. No explanations, no commands, and no other text.

## Final Output Requirement

Output 2–4 separate code blocks (depending on which components are present) to enable easy copy/paste of the individual parts.

Block 1 — type/scope (choose one):
```text
<type />(<scope />):
```
OR
```text
<type />:
```

Block 2 — description:
```text
<description />
```

Block 3 — body (optional):
```markdown
<body />
```

Block 4 — footer (optional):
```text
<footer />
```

Rules:

- If there is no scope, use just `type:`
- Always put the `description` in a block by itself for copy/paste flexibility.
- If there is no body or footer, omit those blocks and output only the header block.
- Use present tense: `add`, not `added`.
- Use imperative mood: `fix bug`, not `fixes bug`.
- Keep the description under 72 characters.
