---
name: pr
description: 'Prompt and workflow for generating a PR title and a structured Mardown PR body by comparing the current branch against the main branch, grouping changes by Conventional Commit types, extracting meaningful notes from commit bodies, and listing issue references in a dedicated “Resolves” section.'
model: GPT-4.1 (copilot)
tools: ['execute/runInTerminal', 'execute/getTerminalOutput']
---

# Instructions

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

- feat: new feature(listed as `feat` in the commit for brevity, but should always be written out full for PR body: `feature`
- fix: bug fix
- docs: documentation only changes
- style: formatting, white-space, etc. (no code meaning changes)
- refactor: code change that neither fixes a bug nor adds a feature
- perf: performance improvements
- test: adding/correcting tests
- build: changes to build system or external dependencies
- ci: changes to CI configuration files and scripts
- revert: reverts a previous commit

BREAKING CHANGE: indicated by a trailing ! after type/scope or a footer/body line starting with “BREAKING CHANGE:”.

## Workflow

1. Determine the current branch:

- Run: `git rev-parse --abbrev-ref HEAD`

2. Determine the base branch:

- If the users specifies a different default branch name, use it.
- If main does not exist, run: `git branch --list`
    - Prefer master if present; otherwise consider trunk or develop if they exist.
    - If more than one plausible base is found, ask the user to choose the base branch before proceeding.

3. Collect commits unique to the current branch since its last common ancestor with the base (exclude merge commits):

- Run: `git log --no-merges --pretty=format:%H%x00%s%x00%b BASE..HEAD`
- Replace BASE with the resolved base branch.
- This must return per commit: SHA, subject, and body separated by NULs (%x00).

4. Parse commit messages using Conventional Commits to identify type, optional scope, and subject:

- Pattern: `type(scope)!: subject`
- Recognize types listed above; unknown types go to other.
- Detect breaking changes if type/scope has a trailing ! or the body contains a line beginning with “BREAKING CHANGE:”.

5. Collect commits diffs from current branch since its last common ancestor with the base:

- Run: `git diff BASE..HEAD`
- Replace `BASE` with the resolved base branch, usually `main`

6. Extract meaningful and important change notes from commit bodies and what you observe in the diff:

- Use concise, user-relevant lines from bodies (prefer bullet-like lines starting with -, *, or short sentences).
- Skip boilerplate and noise.

7. Extract issue references from commit bodies:

- Match patterns like closes #123, fixes #456, resolved/resolves #789.
- Collect unique issue numbers and prepare a “Resolves the following issues:” section with a bulleted list of just the numbers (e.g., “- #123”). Do not include titles or extra text; GitHub will resolve them.

8. Compose the PR content:

- Title:
    - Neutral, concise headline summarizing the main purpose/reason for the PR inferred from the commits (subjects and bodies).
    - 72 characters max, sentence case, no trailing period, no Conventional Commit prefixes.
- Body:
    - One or two sentences that explain the purpose/impact of the changes.
    - A “Changes” section grouped by type in this order: feature, fix, perf, refactor, docs, test, build, ci, style.
        - For each commit, create a bullet primarily from the subject; append brief, relevant details from the body where helpful.
        - If scope is present, you may format as “scope: subject” for clarity.
        - DO NOT include empty type  sections
    - If present, a “Breaking changes” section listing extracted breaking change notes.
    - If present, a “Resolves the following issues:” section as specified above.

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

## feature
- scope: subject — optional concise detail from body

## fix
- scope: subject — optional concise detail from body

## perf
- subject — detail

## refactor
- subject — detail

## docs
- subject — detail

## test
- subject — detail

## build
- subject — detail

## ci
- subject — detail

## style
- subject — detail

## revert
- subject — detail

## Resolves the following issues:
- #123
- #456
```

**Remember:**
- Only include sections that have content (omit empty type sections)
- Omit "Breaking changes" if none exist
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

## feature
- auth: simplify login flow — remove redundant redirects and unify error surfaces

## fix
- auth: handle token refresh edge cases — retry logic added for intermittent network failures

## docs
- update README — add auth troubleshooting section and clearer setup steps

## Resolves the following issues:
- #123
- #456
```
