# Marketplace Reference Unification & `bumping-version` Skill Generalization

**Date:** 2026-04-27
**Status:** Approved (design)
**Repos affected:** `skill-set`, `knowledge-distillery`, `hotwire-frontend-skills` (two branches), `ether-plugins` (this hub)

## Background

`ether-plugins` is a hub marketplace that aggregates several per-plugin repos under one entry point (`claude plugin marketplace add ether-moon/ether-plugins`). Each plugin still ships its own per-plugin `marketplace.json` for backward compatibility (existing single-plugin installs).

Two related cleanups surfaced:

1. **Self-referential dogfooding settings.** Plugin repos register *their own* per-plugin marketplace inside `.claude/settings.json` for in-repo Claude Code development. With the hub now stable, these references should point at the hub.

2. **Duplicated `bumping-version` skill.** `knowledge-distillery` and `hotwire-frontend-skills` (on its `hotwire-skills-integration` branch) each carry a near-identical `bumping-version` skill that drifted (different push behavior, "cc-plugin-template" string left in one description, slightly different file targets). It is a maintenance liability and an obvious candidate for a single shared implementation.

## Goals

- Every plugin repo's dogfooding `settings.json` registers the **hub** marketplace (`ether-plugins`) and enables `<plugin>@ether-plugins`.
- `bumping-version` lives in **one place** (`skill-set`) and is general enough to use in any OSS project — not only Claude plugins.
- Per-repo policy (push behavior, base branch, extra version files) is supplied as **context** in each repo's `CLAUDE.md` / `AGENTS.md`, not embedded in the skill.
- Per-plugin `marketplace.json` files remain (backward-compat policy from `ether-plugins/README.md`). This change touches consumer-side settings only.
- Future plugins generated from `cc-plugin-template` start out hub-aligned by default.

## Non-goals

- Removing per-plugin `marketplace.json` files.
- Building any cross-plugin orchestration (e.g., bumping all plugins from the hub).
- Touching `agent-atelier` or `herb-lsp-plugin` — neither has an established plugin layout / settings.json yet. Out of scope; revisit later.
- Changing how plugins are *published* — `git-subdir` references in `ether-plugins/.claude-plugin/marketplace.json` are unchanged.

## Architecture

```
NEW    skill-set/plugins/skill-set/skills/bumping-version/SKILL.md
         ↑ Single source of truth. Convention-based auto-detect.
         ↑ Repo policy supplied via `## Versioning` section in CLAUDE.md/AGENTS.md.

NEW    skill-set/.claude/settings.json
         (only `.claude/settings.local.json` exists today; create the shared one)
         + extraKnownMarketplaces.ether-plugins (github: ether-moon/ether-plugins)
         + enabledPlugins["skill-set@ether-plugins"]

EDIT   knowledge-distillery/.claude/settings.json
         - extraKnownMarketplaces.knowledge-distillery
         + extraKnownMarketplaces.ether-plugins
         - enabledPlugins["knowledge-distillery@knowledge-distillery"]
         + enabledPlugins["knowledge-distillery@ether-plugins"]
         + enabledPlugins["skill-set@ether-plugins"]
DELETE knowledge-distillery/.claude/skills/bumping-version/

EDIT   hotwire-frontend-skills/.claude/settings.json   (hotwire-skills-integration branch)
         (same shape as knowledge-distillery)
         + enabledPlugins["skill-set@ether-plugins"]
DELETE hotwire-frontend-skills/.claude/skills/bumping-version/   (same branch)

EDIT   hotwire-frontend-skills (main branch — cc-plugin-template)
         - init.sh: add "Hub marketplace name" prompt (default `ether-plugins`, blank → per-plugin fallback)
         - templates/.claude/settings.json: emit hub-style block when hub is provided
         - templates/CLAUDE.md / AGENTS.md: include empty `## Versioning` placeholder + skill-set enable note

NO CHANGE  ether-plugins/.claude-plugin/marketplace.json
ADD        ether-plugins/README.md: one-line note that each plugin's settings.json registers the hub for development.
```

## `bumping-version` skill specification

### Frontmatter

```yaml
---
name: bumping-version
description: Bump a project's version with changelog update — auto-detects version files (plugin.json, package.json, pyproject.toml, Cargo.toml, gemspec, VERSION, ...). Use when user says "bump", "release", "version up". Repo-specific policy (base branch, extra files, commit message) read from CLAUDE.md / AGENTS.md.
---
```

### Workflow

**0. Preflight & worktree setup**

- Determine base branch:
  1. From repo context `## Versioning` → `Base branch:` line, if present.
  2. Otherwise `git symbolic-ref refs/remotes/origin/HEAD` (e.g., `refs/remotes/origin/main`).
- `git fetch origin <base>`.
- Create a temporary worktree on the latest base:
  - Path: `.git/worktrees/bump-<unix-timestamp>` (gitignored by git itself).
  - `git worktree add <path> origin/<base>`
- All subsequent steps run inside the worktree. The user's original checkout is untouched.

**1. Analyze changes**

Two separate Bash calls (no `$(...)` substitution — Claude Code blocks it):

```bash
# Call 1: find last bump commit on base
git -C <worktree> log --oneline --grep="^chore: bump\|^chore: release" -1 --format="%H" origin/<base>
```

If no match → use the first commit on base. Then:

```bash
# Call 2: list commits since last bump (replace <SHA>)
git -C <worktree> log --oneline <SHA>..origin/<base>
```

This intentionally reads from `origin/<base>`, so the changelog only reflects merged commits — never in-flight feature-branch work.

**2. Load repo context**

Read the `## Versioning` section (case-insensitive header match) from `CLAUDE.md` first, then `AGENTS.md`. Recognized keys (all optional):

- `Base branch:` — overrides auto-detection.
- `Commit message:` — template; `{version}` placeholder replaced. Default: `chore: bump version to {version}`.
- `Extra version files:` — comma- or list-separated paths to also update (in addition to auto-detected ones).
- `Changelog categories:` — override defaults (`Added`, `Improved`, `Fixed`).

If the section is missing, all defaults apply. The skill is usable in a brand-new repo with zero context.

**3. Detect version files & current version**

Scan in order, list every match found:

| Ecosystem | File pattern | Field |
|---|---|---|
| Claude plugin | `**/.claude-plugin/plugin.json` | `.version` (JSON) |
| Node | `package.json` | `.version` (JSON) |
| Python (modern) | `pyproject.toml` | `[project].version` or `[tool.poetry].version` |
| Python (legacy) | `**/__init__.py` | line `__version__ =` |
| Rust | `Cargo.toml` | `[package].version` |
| Ruby | `*.gemspec`, `lib/**/version.rb` | `spec.version =` / `VERSION =` |
| Generic | `VERSION`, `version.txt` (root only) | whole file (one line) |
| Repo-specified | from `Extra version files:` | matched by regex `version[:= ]+"?(\d+\.\d+\.\d+)"?` |

Display the discovered files with their current version.

**4. Verify version consistency**

All discovered files must show the same current version. If any differ, stop and ask which to take as truth before proceeding.

**5. Suggest semver level**

Based on the commit list, recommend `patch` / `minor` / `major` with one-line rationale, then ask the user to choose `[1] patch` / `[2] minor` / `[3] major`. Use the user's language for the prompt.

**6. Update files**

Apply new version to every discovered manifest. For changelog (`CHANGELOG.md` at repo root):

- Format: Keep a Changelog (`## [X.Y.Z] - YYYY-MM-DD`).
- New entry prepended after the file header, before the previously-latest entry.
- Categories from context (default `Added` / `Improved` / `Fixed`).
- Date is today (in UTC; matches existing skill behavior).

If `CHANGELOG.md` is absent, ask whether to create it; declining skips the changelog step.

**7. Commit**

Inside worktree: stage all updated files + changelog, create a single commit. Message uses repo context template (default `chore: bump version to X.Y.Z`).

**8. Push or PR fallback**

```
1st: git -C <worktree> push origin HEAD:<base>
     ├─ success → done
     └─ fail
        ├─ matches protection pattern (regex: GH006|protected branch|remote rejected|hook declined)
        │   ├─ git -C <worktree> branch bump-version-X.Y.Z
        │   ├─ git -C <worktree> push -u origin bump-version-X.Y.Z
        │   └─ gh pr create --base <base> \
        │           --head bump-version-X.Y.Z \
        │           --title "chore: bump version to X.Y.Z" \
        │           --body "<changelog excerpt>"
        │      → emit PR URL to user
        └─ other failure → report error, stop, leave worktree for inspection
```

**9. Cleanup**

- On clean push success: `git worktree remove <path>`.
- On PR fallback: `git worktree remove <path>` (the branch lives on `origin`, no longer needs the local worktree).
- On unrecoverable error: leave worktree, print its path, exit.
- The user's original branch and working tree are unchanged throughout.

### Rules

- Bump commit is always its own commit (never mixed with other work).
- Changelog date is today.
- Version inconsistency across manifests blocks automatic progress — always ask.
- Never `--force` push.

### Repo-context interface (example)

```markdown
## Versioning

- **Base branch**: main
- **Commit message**: chore: bump version to {version}
- **Extra version files**: docs/install.md
- **Changelog categories**: Added, Improved, Fixed
```

## Marketplace settings.json target shape

For each plugin repo (using `knowledge-distillery` as illustration). The existing `permissions` block is preserved as-is; only `extraKnownMarketplaces` and `enabledPlugins` change:

```json
{
  "permissions": {
    "allow": ["...preserved as-is..."]
  },
  "extraKnownMarketplaces": {
    "ether-plugins": {
      "source": {
        "source": "github",
        "repo": "ether-moon/ether-plugins"
      }
    }
  },
  "enabledPlugins": {
    "knowledge-distillery@ether-plugins": true,
    "skill-set@ether-plugins": true
  }
}
```

`skill-set` is enabled in every plugin repo so its `bumping-version` (and other tools) is available during development.

## `cc-plugin-template` changes

`init.sh` adds one new prompt:

| Prompt | Example | Used for |
|---|---|---|
| Hub marketplace name (optional) | `ether-plugins` | If non-empty, generated `.claude/settings.json` registers the hub and enables `<plugin>@<hub>`. If empty, falls back to current per-plugin self-registration behavior. |

The generated `CLAUDE.md` includes:

```markdown
## Versioning

<!-- Filled in per project. See `bumping-version` skill in skill-set. -->
- **Base branch**:
- **Commit message**:
- **Extra version files**:
- **Changelog categories**:
```

And the generated `.claude/settings.json` includes `skill-set@<hub>` in `enabledPlugins` when a hub is provided.

## Migration order

Each phase is one or more PRs in the listed repo. Phases must complete in order; PRs within a phase can be parallel.

**Phase 1 — Tooling lands first**
1. `skill-set` repo
   - Add `plugins/skill-set/skills/bumping-version/SKILL.md` per the spec above.
   - Add `## Versioning` section to `skill-set`'s own `CLAUDE.md` / `AGENTS.md`.
   - Add the hub dogfooding block to `.claude/settings.json` (currently absent).
   - Bump `skill-set` itself using the new skill (self-test).

**Phase 2 — Consumer repos** (after Phase 1 release)

2. `knowledge-distillery` repo
   - Update `.claude/settings.json` per the target shape above.
   - Move the per-plugin policy from the old skill (push to main, `plugin.json` is the version source) into a new `## Versioning` section in `CLAUDE.md` / `AGENTS.md`.
   - Delete `.claude/skills/bumping-version/`.

3. `hotwire-frontend-skills` repo, `hotwire-skills-integration` branch
   - Same changes as (2), with its own policy values (no extra version files, default push behavior).
   - Delete `.claude/skills/bumping-version/`.

**Phase 3 — Template alignment** (parallel to Phase 2 OK)

4. `hotwire-frontend-skills` repo, `main` branch (= `cc-plugin-template`)
   - Add hub prompt to `init.sh`.
   - Update template `.claude/settings.json` to emit hub-style block when hub is provided.
   - Add `## Versioning` placeholder to template `CLAUDE.md` / `AGENTS.md`.

**Phase 4 — Hub note**

5. `ether-plugins` repo (this hub)
   - One-line addition to README clarifying the dogfooding pattern.
   - No change to `marketplace.json`.

## Risks & mitigations

- **`gh` not authenticated when PR fallback triggers.** Skill must detect missing `gh` and instruct the user to run `gh auth login` or push manually; do not silently fail.
- **Worktree leaks on crash.** Document the worktree path in every error message so the user can `git worktree remove` manually.
- **Version regex false positives in `Extra version files`.** Conservative regex (`version[:= ]+"?(\d+\.\d+\.\d+)"?`); skill confirms intended replacement before applying.
- **Hub not yet aware of a brand-new plugin.** Dogfooding via hub will fail until the plugin is added to `ether-plugins/.claude-plugin/marketplace.json`. This is a useful forcing function — flag it in the README.
