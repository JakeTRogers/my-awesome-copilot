---
name: pr
description: 'Prompt and workflow for generating a PR title and a structured Markdown PR body by comparing the current branch against the main branch, grouping changes by Conventional Commit types, extracting meaningful notes from commit bodies, and listing issue references in a dedicated “Resolves” section.'
agent: agent
argument-hint: '[optional-base-branch]'
model: GPT-5.4 mini (copilot)
tools: [vscode/askQuestions, execute/getTerminalOutput, execute/runInTerminal, read/readFile, search]
---

# Generate Pull Request

## Critical Rules

1. **ALWAYS run all required git commands** - This is mandatory, not optional
2. **Execute commands in the exact order specified** in the Workflow section
3. **When this prompt is triggered again**, re-run all commands from the beginning to get current state
4. **Output format**: Provide exactly 2 code blocks as specified - no additional text, explanations, or commands

## Content Guidelines

- Use commit subjects and bodies to generate the PR content
- Title: neutral, concise, up to 72 characters, no trailing period, no Conventional Commit prefixes
- Body: grouped markdown header sections by Conventional Commit type, optional "Breaking changes," and a "Resolves the following issues" sections listing only issue numbers (e.g., - #123)

## Conventional Commits Types

- feat: new feature; render this type under the `Feature` section in the PR body
- fix: bug fix
- docs: documentation only changes
- style: formatting, white-space, etc. (no code meaning changes)
- refactor: code change that neither fixes a bug nor adds a feature
- perf: performance improvements
- test: adding/correcting tests
- build: changes to build system or external dependencies
- ci: changes to CI configuration files and scripts
- revert: reverts a previous commit

Unknown or non-standard commit types should be grouped under an `Other` section.

BREAKING CHANGE: indicated by a trailing ! after type/scope or a footer/body line starting with “BREAKING CHANGE:”.

## Workflow

1. Determine the current branch:

  - Run: `git rev-parse --abbrev-ref HEAD`

2. Determine the base branch:

  - If the user specifies a different default branch name, use it.
  - If main does not exist, run: `git branch --list`
    - Prefer master if present; otherwise consider trunk or develop if they exist.
    - If more than one plausible base is found, use the `vscode/askQuestions` tool to ask the user to choose the base branch before proceeding.

3. Collect commits unique to the current branch relative to the base (exclude merge commits):

  - Run: `git log --no-merges --pretty=format:%H%x00%s%x00%b BASE..HEAD`
  - Replace BASE with the resolved base branch.
  - This must return per commit: SHA, subject, and body separated by NULs (%x00).
  - If this command returns no commits, use the `vscode/askQuestions` tool to ask whether the user wants to compare against a different base branch before proceeding.

4. Parse commit messages using Conventional Commits to identify type, optional scope, and subject:

  - Pattern: `type(scope)!: subject`
  - Recognize types listed above; unknown types go to `Other`.
  - Detect breaking changes if type/scope has a trailing ! or the body contains a line beginning with “BREAKING CHANGE:”.

5. Collect commits diffs from current branch since its last common ancestor with the base:

  - Run: `git diff BASE...HEAD`
  - Replace `BASE` with the resolved base branch, usually `main`.
  - This must compare `HEAD` against the merge-base with `BASE`, not against the current tip of `BASE`.

6. Extract meaningful and important change notes from commit bodies and what you observe in the diff:

  - Use concise, user-relevant lines from bodies (prefer bullet-like lines starting with -, *, or short sentences).
  - Skip boilerplate and noise.
  - Prefer impact-oriented notes over file-by-file implementation details.
  - Use the read/readFile and search tools to gather additional context from relevant files if needed to understand the impact of changes and extract meaningful notes.

7. Extract issue references from commit bodies:

  - Match patterns like close/closes/closed #123, fix/fixes/fixed #456, and resolve/resolves/resolved #789.
  - Collect unique issue numbers and prepare a “Resolves the following issues:” section with a bulleted list of just the numbers (e.g., “- #123”). Do not include titles or extra text; GitHub will resolve them.

8. Compose the PR content:

  - Title:
    - Neutral, concise headline summarizing the main purpose/reason for the PR inferred from the commits (subjects and bodies).
    - 72 characters max, sentence case, no trailing period, no Conventional Commit prefixes.
  - Body:
    - One or two sentences that explain the purpose/impact of the changes.
    - A “Changes” section grouped by type in this order: Feature, Fix, Performance, Refactor, Docs, Test, Build, CI, Style, Revert, Other.
      - For each commit, create a bullet primarily from the subject; append brief, relevant details from the body where helpful.
      - If scope is present, you may format as “scope: subject” for clarity.
      - If one commit spans multiple concerns, place it under the single best-fit section instead of repeating it.
      - DO NOT include empty type sections
    - If present, a “Breaking changes” section listing extracted breaking change notes.
    - If present, a “Resolves the following issues:” section as specified above.

## Quality Checks

- Ensure the title and summary reflect the branch-wide purpose, not just the latest commit.
- Remove repetitive bullets caused by fixup or follow-up commits when the net change can be described once.
- Prefer user-facing behavior, risk, and migration impact over low-level implementation trivia.
- If the diff materially changes behavior that is not obvious from commit messages alone, incorporate that behavior into the body.

## Final Output Format

**CRITICAL**: Your response must contain ONLY the 2 code blocks below. No explanations, no git commands, no other text.

**Block 1 — PR Title** (plain text):

```text
Your concise PR title here
```

**Block 2 — PR Body** (markdown):

```markdown
One–two sentence overview of purpose/impact.

# Changes

## Breaking changes
- Breaking change note(s) if any

## Feature
- scope: subject

## Fix
- scope: subject

## Performance
- scope: subject

## Refactor
- scope: subject

## Docs
- scope: subject

## Test
- scope: subject

## Build
- scope: subject

## CI
- scope: subject

## Style
- scope: subject

## Revert
- scope: subject

## Other
- scope: subject

## Resolves the following issues:
- #123
- #456
```

**Remember:**
- Only include sections that have content (omit empty type sections)
- Omit "Breaking changes" if none exist
- If there are breaking changes, list them as the first section in the body, before features and fixes
- Omit "Resolves the following issues" if no issues are referenced

## Examples

Example PR Title:

```text
Improve authentication reliability and user experience
```

Example PR Body:

```markdown
Streamlines the login flow, hardens error handling, and clarifies auth-related messaging to reduce friction and failures.

# Changes

## Breaking changes
- auth: remove support for legacy token exchange endpoint, clients must now use the new unified auth endpoint, which may require updates to custom integrations

## Feature
- auth: simplify login flow, remove redundant redirects and unify error surfaces

## Fix
- auth: handle token refresh edge cases, retry logic added for intermittent network failures

## Docs
- add auth troubleshooting section and clearer setup steps to README

## Resolves the following issues:
- #123
- #456
```
