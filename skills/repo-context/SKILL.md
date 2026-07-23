---
name: repo-context
description: 'Discover a repository and generate or refresh AGENTS.md as the canonical AI context file shared by GitHub Copilot and Claude Code, with CLAUDE.md as a pointer. Use when asked to document a repo for AI agents, build repo context, create or update AGENTS.md, onboard an agent to a codebase, or before starting a new feature in an undocumented repository.'
---

# Repo Context

## Purpose

Produce one durable, canonical context file — `AGENTS.md` at the repository root — that both GitHub Copilot and Claude Code load when starting work in the repo, so every new feature or session begins with accurate knowledge of the build commands, architecture, and conventions. Copilot reads `AGENTS.md` natively; Claude Code reads it through a `CLAUDE.md` pointer.

Two modes:

- **Generate** — no `AGENTS.md` exists: discover the repo and write it.
- **Refresh** — `AGENTS.md` exists: update only the sections that drifted since it was last touched.

## Constraints

- `AGENTS.md` is loaded into every agent session, so keep it under ~150 lines. Push deep detail into `docs/` and link to it instead of inlining.
- Record only what an agent cannot cheaply derive by reading the code: commands, conventions, non-obvious relationships, and gotchas. Do not paraphrase source files.
- Never create commits or tags. Write the files and hand the user a suggested conventional commit command (the user signs commits with a YubiKey).

## Mode 1: Generate

### 1. Discover

Inventory the repo before writing anything:

```bash
git ls-files | cut -d/ -f1 | sort | uniq -c | sort -rn   # directory map
git log --oneline -15                                     # recent activity
```

The histogram shows the shape of a repo at any size — an alphabetical file dump does not. Drill into the large directories with `git ls-files <dir>` where the top level is not enough.

Then inspect, where present: `README.md`, `Makefile`/`Taskfile.yml`/`justfile`, `package.json` scripts, `go.mod`, `pyproject.toml`, `.pre-commit-config.yaml`, `.cz.yaml`, CI workflows under `.github/workflows/`, and any existing `CLAUDE.md`, `.github/copilot-instructions.md`, or `docs/`. These reveal the real build/test/release commands and conventions — prefer them over guessing.

If existing AI-context files are found (`CLAUDE.md` with content, `.github/copilot-instructions.md`), fold their still-accurate content into `AGENTS.md` and propose replacing them with pointers, but ask the user before deleting anything you did not create.

### 2. Write AGENTS.md

Use this structure, omitting sections that do not apply:

```markdown
# <Project Name>

<One or two sentences: what this project is and who uses it.>

## Commands

- Build: `<command>`
- Test: `<command>` (single test: `<command>`)
- Lint/format: `<command>`
- Run locally: `<command>`

## Architecture

<Short map: top-level directories and what lives in each, key entry points,
and the one or two non-obvious relationships worth knowing. Link to
docs/<topic>.md for anything deeper.>

## Conventions

- <Code style: pointer to the applicable instructions file, linter config, or dominant pattern.>
- <Testing conventions: where tests live, naming, fixtures.>

## Gotchas

- <Things that will waste an agent's time if unknown: slow test suites, generated files, ordering requirements, env vars.>
```

### 3. Bridge Claude Code

Create `CLAUDE.md` at the repo root containing only an import of the canonical file:

```markdown
@AGENTS.md
```

If a meaningful `CLAUDE.md` already exists, merge its unique content into `AGENTS.md` first, then reduce `CLAUDE.md` to the pointer.

### 4. Verify

Before handing off:

- Trace every command in `AGENTS.md` back to its source — a Makefile target, `package.json` script, CI step, or documented invocation. Fix or drop any command you cannot trace; a wrong command costs an agent more time than a missing one.
- Check the size with `wc -l AGENTS.md`. If it exceeds ~150 lines, move detail into `docs/` and link to it.

### 5. Hand off

Suggest a commit for the user to run — never run it yourself — matching the commit conventions observed in `git log`, e.g.:

```bash
git add AGENTS.md CLAUDE.md
git commit -m "docs: add AGENTS.md as canonical agent context"
```

## Mode 2: Refresh

1. Find what changed since the file was last updated:

   ```bash
   last=$(git log -1 --format=%H -- AGENTS.md)
   git log --oneline --stat "${last}..HEAD"
   ```

   If `$last` is empty, `AGENTS.md` exists but was never committed, and there is no baseline to diff against — the range above degrades to `..HEAD`, which prints nothing and would falsely suggest the file is current. In that case skip the drift comparison and audit every section directly against the repo, as in Generate step 4.

2. For each AGENTS.md section, check whether the changes invalidate it: new or renamed directories (Architecture), changed Makefile/CI/package scripts (Commands), new tooling or lint config (Conventions), new generated files, env vars, or ordering requirements (Gotchas).
3. Update only stale sections; preserve wording of sections that are still accurate so diffs stay reviewable.
4. Verify every command still exists in the repo's own files before keeping it.
5. If nothing is stale, say so and change nothing.
6. Hand off with a suggested `docs: refresh AGENTS.md` commit command as in Generate step 5.

## Monorepos

The agents.md spec supports nested `AGENTS.md` files: agents use the closest file above their working directory. Keep the root `AGENTS.md` for the cross-cutting picture — workspace layout, shared commands, repo-wide conventions — and add `<package>/AGENTS.md` only where a package's commands or conventions genuinely diverge from the root. Give each nested file a sibling `CLAUDE.md` containing `@AGENTS.md`, same as the root bridge. A nested file that repeats the root is drift waiting to happen; the same line budget applies to each file.

## When NOT to Use This Skill

- For personal/global preferences — those belong in user-level config, not the repo.
- To duplicate what `README.md` already says for humans — link to it instead.
- To document code that is about to be rewritten; refresh after the change lands.
