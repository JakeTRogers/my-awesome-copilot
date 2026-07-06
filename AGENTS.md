# my-awesome-copilot

Personal collection of GitHub Copilot / Claude Code prompts, agents, instructions, and skills — a public notebook of patterns actually in use, not an awesome-list fork.

## Commands

- Validate everything: `pre-commit run --all-files` (large files, EOF, merge conflicts, JSON/YAML, trailing whitespace, commit-msg format)
- There is no build or test suite; the artifacts are markdown files consumed by AI tooling.

## Architecture

- `prompts/` — VS Code prompt files (`*.prompt.md`), e.g. `cc` (conventional commit) and `pr` (pull request). Authoring guide: `.github/instructions/prompt.instructions.md`.
- `skills/` — portable Agent Skills (`<name>/SKILL.md`) usable by both Copilot and Claude Code. Authoring guide: `.github/instructions/agent-skills.instructions.md`.
- `agents/` — a Maestro-orchestrated multi-agent system (`*.agent.md`): Architect, Auditor, Chronicler, Coder, Researcher, Tester, plus the standalone Promptly prompt-engineering mode. Authoring guide: `.github/instructions/agents.instructions.md`.
- `instructions/` — language/format conventions applied by glob (`applyTo` frontmatter): Go, Python, shell, markdown.

## Conventions

- Commits: Conventional Commits enforced by commitizen (`.cz.yaml`) + pre-commit `commit-msg` hook. Never use the `chore` type; never use multiple scopes. Commits are GPG-signed by the user via YubiKey — agents must never create commits or tags; hand the user the commands instead.
- Markdown: follow `instructions/markdown.instructions.md` — blank lines around headings/lists/fences, language on every fence, no bare URLs.
- Skill frontmatter must stay portable across Copilot and Claude Code (agentskills.io spec); descriptions must state WHAT the skill does and WHEN to use it.
- Prompts are self-contained (they must work where personal skills are not installed) but their rules must stay content-aligned with the corresponding skill.

## Gotchas

- Skill bodies embedding markdown templates that contain code fences must use 4-backtick outer fences to avoid premature fence termination. Otherwise use standard 3-backtick fences.
