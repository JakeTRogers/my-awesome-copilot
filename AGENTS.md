# my-awesome-copilot

Personal collection of GitHub Copilot / Claude Code instructions and skills — a public notebook of patterns actually in use, not an awesome-list fork.

## Commands

- Verify: `pre-commit run --all-files`. There is no build or test suite, so a change is done when pre-commit passes and the issue's acceptance checks hold.
- Skill discovery: `gh skill install . --from-local` lists every skill users can install.

## Architecture

- `skills/<name>/SKILL.md` — portable Agent Skills for Copilot and Claude Code. Users install them one at a time with `gh skill install`, so each skill is self-contained: it links only to files in its own folder.
- `instructions/` — `applyTo` convention files for other repos to install. No host loads them in this repo.

## Conventions

- Commits: Conventional Commits, checked by commitizen (`.cz.yaml`) in the pre-commit `commit-msg` hook. One scope at most. Every commit gets the type that names its change (`build` for tooling and config, `docs` for prose); `chore` is unused because it becomes a junk drawer. Write messages with `skills/conventional-commit`.
- Releases: every PR carries a `cz bump` commit (CI checks the version in `.cz.yaml`), which regenerates `CHANGELOG.md` and creates a signed tag. The version, `CHANGELOG.md`, and tags belong to the maintainer.
- Signing and history: the maintainer GPG-signs every commit via YubiKey and keeps history linear.
  - In a `.worktrees/<name>` worktree, on a local branch cut from the start branch: commit and rebase unsigned (`git -c commit.gpgsign=false commit|rebase`), rebase onto the start branch's tip, and land by fast-forward. The work stays local; remove the worktree and its branch once landed.
  - Fixups, in a worktree or not: commit them unsigned (`git -c commit.gpgsign=false commit --fixup=<sha>`). The maintainer's `gresign` folds them in and re-signs the branch in one pass.
  - Outside a worktree, any other change: stage it and leave committing to the maintainer.
  - After work lands or fixups are committed, tell the maintainer to run `gresign`. Pushing and pull requests are theirs.
- Markdown: follow `instructions/markdown.instructions.md` — blank lines around headings/lists/fences, language on every fence, no bare URLs.
- Skills: frontmatter follows the agentskills.io spec, plus host-specific keys such as `disable-model-invocation` where a host needs them; the description states WHAT the skill does and WHEN to use it. A skill adapted from elsewhere credits its source three ways: `metadata.upstream` in frontmatter, an "Adapted from … at `<version>`" line, and an entry in the README's Credits.

## Gotchas

- Skill bodies embedding markdown templates that contain code fences need 4-backtick outer fences, or the inner fence ends the outer one.

## Agent skills

### Issue tracker

GitHub Issues on `JakeTRogers/my-awesome-copilot`, via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five default triage labels, each label string the same as its role name. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `GLOSSARY.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
