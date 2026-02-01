---
name: cc
description: 'Prompt and workflow for generating conventional commit messages using a structured XML format. Guides users to create standardized, descriptive commit messages in line with the Conventional Commits specification, including instructions, examples, and validation.'
model: GPT-4.1 (copilot)
tools: ['execute/runInTerminal', 'execute/getTerminalOutput']
---

# Instructions

## Critical Rules

1. **ALWAYS run both commands at the start** - This is mandatory, not optional:
   - First: `git status`
   - Second: `git diff --cached`
2. **When this prompt is triggered again in the same session**, the user has staged different/additional files. You MUST re-run both commands from step 1, even if you ran them before.
3. If multiple files are staged, generate a single commit message that identifies the primary reason for the changes.

## Conventional Commits Types

- fix: a commit of the type fix patches a bug in your codebase (this correlates with PATCH in Semantic Versioning).
- feat: a commit of the type feat introduces a new feature to the codebase (this correlates with MINOR in Semantic Versioning).
- docs: documentation only changes
- style: Changes that do not affect the meaning of the code (white-space, formatting, missing semi-colons, etc)
- refactor: A code change that neither fixes a bug nor adds a feature
- perf: A code change that improves performance
- test: Adding missing or correcting existing tests
- build: Changes that affect the build system or external dependencies (example scopes: pip, docker, npm)
- ci: Changes to CI configuration files and scripts (example scopes: GitLabCI)

BREAKING CHANGE: a commit that has a footer, BREAKING CHANGE:, or appends a ! after the type/scope, introduces a breaking API change (correlating with MAJOR in Semantic Versioning). A BREAKING CHANGE can be part of commits of any type.

## Workflow

**Execute these steps in order - no exceptions:**

1. **MUST RUN**: Execute `git status` in the terminal to review changed files
2. **MUST RUN**: Execute `git diff --cached` in the terminal to inspect staged changes
3. **Analyze** the output from both commands to understand what changed and why
4. **Construct** the commit message with these components:
   - **type** (required): One of: feat, fix, docs, style, refactor, perf, test, build, ci
   - **scope** (optional): Component/area affected
   - **description** (required): Short imperative summary
   - **body** (optional): Additional context only if description isn't sufficient
   - **footer** (optional): Breaking changes or issue references
5. **Output** the commit message in the exact format specified in "Final Output Format" section below



## Example Commit Types

- `feat(parser): add ability to parse arrays`
- `fix(ui): correct button alignment`
- `docs: update README with usage instructions`
- `refactor: improve performance of data processing`
- `feat!: send email on registration`

## Validation

```xml
<validation>
  <type>Must be one of the allowed types.</type>
  <scope>Optional, but recommended for clarity.</scope>
  <description>Required. Use the imperative mood (e.g., "add", not "added").</description>
  <body>Optional. Use for additional context, but only if the description is not enough.</body>
  <footer>Use for breaking changes or issue references.</footer>
</validation>
```

## Final Output Format

**CRITICAL**: Your response must contain ONLY the commit message blocks below. No explanations, no XML, no commands, no other text.

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
```text
<body />
```

Block 4 — footer (optional):
```text
<footer />
```

**Rules:**
- If no scope: `type: description`
- If no body or footer: omit those lines (just output `type: description`)
- Present tense: "add" not "added"
- Imperative mood: "fix bug" not "fixes bug"
- Keep description under 72 characters
