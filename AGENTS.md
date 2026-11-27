# my-awesome-copilot

Personal collection of GitHub Copilot / Claude Code agents, instructions, and skills — a public notebook of patterns actually in use, not an awesome-list fork.

## Commands

- Validate everything: `pre-commit run --all-files` (large files, EOF, merge conflicts, JSON/YAML, trailing whitespace, commit-msg format)
- There is no build or test suite; the artifacts are markdown files consumed by AI tooling.

## Architecture

- `skills/` — portable Agent Skills (`<name>/SKILL.md`) usable by both Copilot and Claude Code, e.g. `/pull-request`. Authoring guide: `.github/instructions/agent-skills.instructions.md`.
- `agents/` — a Maestro-orchestrated multi-agent system (`*.agent.md`): Architect, Auditor, Chronicler, Coder, Researcher, Tester, plus the standalone Promptly prompt-engineering mode. Authoring guide: `.github/instructions/agents.instructions.md`.
- `instructions/` — language/format conventions applied by glob (`applyTo` frontmatter): Go, Python, shell, markdown.

## Conventions

- Commits: Conventional Commits enforced by commitizen (`.cz.yaml`) + pre-commit `commit-msg` hook. Never use the `chore` type; never use multiple scopes.
- Signing and history: the maintainer GPG-signs every commit via YubiKey and keeps history linear.
  - In a worktree (`.worktrees/<name>`, on a local branch cut from the start branch): commit and rebase unsigned (`git -c commit.gpgsign=false commit|rebase`), rebase onto the start branch's tip, and land by fast-forward only. Never push or open a pull request from a worktree branch; remove the worktree and delete its branch once landed.
  - Outside a worktree: stage changes and leave committing to the maintainer.
  - Never create tags. After work lands, tell the maintainer to re-sign the start branch (`gresign`) before pushing.
- Markdown: follow `instructions/markdown.instructions.md` — blank lines around headings/lists/fences, language on every fence, no bare URLs.
- Skill frontmatter must stay portable across Copilot and Claude Code (agentskills.io spec); descriptions must state WHAT the skill does and WHEN to use it.

## Gotchas

- Skill bodies embedding markdown templates that contain code fences must use 4-backtick outer fences to avoid premature fence termination. Otherwise use standard 3-backtick fences.

## Agent skills

### Issue tracker

GitHub Issues on `JakeTRogers/my-awesome-copilot`, via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five default triage labels, each label string the same as its role name. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `GLOSSARY.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
