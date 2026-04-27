# Marketplace Reference Unification & `bumping-version` Generalization — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Unify all per-plugin dogfooding `settings.json` references onto the `ether-plugins` hub, replace duplicated `bumping-version` skills with a single generalized skill living in `skill-set`, and align the `cc-plugin-template` so future plugins start out hub-aligned. Each repo finishes with a PR.

**Architecture:** 5 task groups across 4 separate plugin repos plus this hub. Phase 1 (`skill-set`) introduces the new skill and must merge first because Phase 2 consumers enable it. Phases 3 and 4 are independent of Phases 1–2.

**Tech Stack:** Bash, `git` worktrees, `gh` CLI, JSON, Markdown. The `bumping-version` skill itself is markdown-only — Claude follows the workflow at runtime; no compiled code.

**Spec:** `docs/superpowers/specs/2026-04-27-marketplace-refs-and-bumping-version-design.md`

**Repo paths:**
- skill-set: `/Users/ether/workspace/personal/skill-set`
- knowledge-distillery: `/Users/ether/workspace/personal/knowledge-distillery`
- hotwire-frontend-skills: `/Users/ether/workspace/personal/hotwire-frontend-skills`
- ether-plugins (this hub): `/Users/ether/conductor/workspaces/ether-plugins-v1/osaka-v2`

**Universal rules for every task:**
- Each repo's PR is opened with `gh pr create --base <repo-default-branch>`. Confirm `gh auth status` is OK before starting.
- Branch naming convention: short, kebab-case, no prefix.
- Commit messages: present tense, `feat:` / `chore:` / `docs:` / `refactor:` prefix.
- Do NOT touch unrelated files in any repo.

---

## Task 1: Phase 1 — `skill-set` repo

**Files:**
- Create: `/Users/ether/workspace/personal/skill-set/plugins/skill-set/skills/bumping-version/SKILL.md`
- Modify: `/Users/ether/workspace/personal/skill-set/AGENTS.md` (add `## Versioning` section)
- Create: `/Users/ether/workspace/personal/skill-set/.claude/settings.json` (currently absent — only `settings.local.json` exists)

**Branch:** `bumping-version-skill`
**PR base:** `main`

- [ ] **Step 1.1: Verify clean working tree and create branch**

```bash
cd /Users/ether/workspace/personal/skill-set
git status                          # expect: clean, on main
git fetch origin
git checkout -b bumping-version-skill origin/main
```

Expected: `Switched to a new branch 'bumping-version-skill'`.

- [ ] **Step 1.2: Create the skill directory and SKILL.md**

```bash
mkdir -p plugins/skill-set/skills/bumping-version
```

Write `plugins/skill-set/skills/bumping-version/SKILL.md` with this exact content:

````markdown
---
name: bumping-version
description: Bump a project's version with changelog update — auto-detects version files (plugin.json, package.json, pyproject.toml, Cargo.toml, gemspec, VERSION, ...). Use when user says "bump", "version up", or "release". Per-repo policy (base branch, extra files, commit message) read from CLAUDE.md / AGENTS.md.
---

# Bumping Version

Bump a project's version, update its changelog, commit on top of the latest base branch, and either push directly or fall back to opening a PR if the base is protected.

The skill is repo-agnostic. Per-repo policy lives in `CLAUDE.md` / `AGENTS.md` under a `## Versioning` section. Defaults apply when the section is absent.

## Workflow

### 0. Preflight & worktree setup

1. **Determine the base branch:**
   - First check the repo's `## Versioning` section for `Base branch:`.
   - Fall back to `git symbolic-ref refs/remotes/origin/HEAD | sed 's|.*/||'` (e.g., `main`).

2. **Sync the base:**

```bash
git fetch origin <base>
```

3. **Create a temporary worktree on the latest base.** This isolates the bump from the user's current checkout — no preflight cleanliness required.

```bash
WT_PATH=".git/worktrees/bump-$(date +%s)"
git worktree add "$WT_PATH" "origin/<base>"
```

All subsequent steps use `git -C "$WT_PATH" ...`.

### 1. Analyze changes since last bump

Run two separate Bash calls (no `$(...)` substitution — Claude Code blocks it):

```bash
# Call 1: find last bump commit on base
git -C "$WT_PATH" log --oneline --grep="^chore: bump\|^chore: release" -1 --format="%H" origin/<base>
```

Read the SHA from the output. If empty, treat the first commit on base as the start. Then:

```bash
# Call 2: list commits since last bump (replace <SHA> with output from Call 1)
git -C "$WT_PATH" log --oneline <SHA>..origin/<base>
```

Reading from `origin/<base>` ensures the changelog reflects only merged commits.

### 2. Load repo context

Read the `## Versioning` section (case-insensitive header match) from `CLAUDE.md` first, then `AGENTS.md`. Recognized keys (all optional):

- `Base branch:` — overrides auto-detection.
- `Commit message:` — template; `{version}` placeholder replaced. Default: `chore: bump version to {version}`.
- `Extra version files:` — comma-separated paths to also update (in addition to auto-detected ones).
- `Changelog categories:` — override defaults (`Added`, `Improved`, `Fixed`).

If the section is missing, all defaults apply.

### 3. Detect version files & current version

Scan in order, list every file found and its current version:

| Ecosystem | File pattern | Field |
|---|---|---|
| Claude plugin | `**/.claude-plugin/plugin.json` | `.version` (JSON) |
| Node | `package.json` | `.version` (JSON) |
| Python (modern) | `pyproject.toml` | `[project].version` or `[tool.poetry].version` |
| Python (legacy) | `**/__init__.py` | line `__version__ =` |
| Rust | `Cargo.toml` | `[package].version` |
| Ruby | `*.gemspec`, `lib/**/version.rb` | `spec.version =` / `VERSION =` |
| Generic | `VERSION`, `version.txt` (root only) | whole file (one line) |
| Repo-specified | from `Extra version files:` | regex `version[:= ]+"?(\d+\.\d+\.\d+)"?` |

Display the discovered files with their current version.

### 4. Verify version consistency

All discovered files must show the same current version. If any differ, stop and ask which to take as truth before proceeding.

### 5. Suggest semver level

Based on the commit list, recommend `patch` / `minor` / `major` with one-line rationale, then ask the user to choose. Use the user's language for the prompt.

```
[In user's language]

Changes since last bump:
- [commit summary 1]
- [commit summary 2]

Current version: X.Y.Z
Suggested: <patch|minor|major> (reason)

Which level?
- [1] patch
- [2] minor
- [3] major
```

### 6. Update files

Apply the new version to every discovered manifest. For changelog (`CHANGELOG.md` at repo root):

- Format: Keep a Changelog (`## [X.Y.Z] - YYYY-MM-DD`).
- New entry prepended after the file header, before the previously-latest entry.
- Categories from context (default `Added` / `Improved` / `Fixed`).
- Date is today.

If `CHANGELOG.md` is absent, ask whether to create it; declining skips the changelog step.

### 7. Commit

Inside worktree: stage all updated files + changelog, create a single commit. Message uses the repo context template (default `chore: bump version to X.Y.Z`).

```bash
git -C "$WT_PATH" add -A
git -C "$WT_PATH" commit -m "chore: bump version to X.Y.Z"
```

### 8. Push or PR fallback

```bash
# 1st attempt: push directly to base
git -C "$WT_PATH" push origin HEAD:<base>
```

If push **succeeds** → proceed to cleanup.

If push **fails**, capture stderr and check for a protection-error pattern (regex `GH006|protected branch|remote rejected|hook declined`). On match, fall back to PR:

```bash
git -C "$WT_PATH" branch bump-version-X.Y.Z
git -C "$WT_PATH" push -u origin bump-version-X.Y.Z
gh -C "$WT_PATH" pr create \
  --base <base> \
  --head bump-version-X.Y.Z \
  --title "chore: bump version to X.Y.Z" \
  --body "$(cat <<'EOF'
## Changelog

<paste the new CHANGELOG entry here>
EOF
)"
```

Emit the PR URL to the user.

If the failure does **not** match the protection pattern (e.g., network error, auth issue), report the error verbatim, leave the worktree for inspection, and stop.

### 9. Cleanup

```bash
git worktree remove "$WT_PATH"
```

Always print the original branch name back to the user so they know nothing changed in their working tree.

## Rules

- Bump commit is always its own commit (never mixed with other work).
- Changelog date is today.
- Version inconsistency across manifests blocks automatic progress — always ask.
- Never `--force` push.
- If `gh` is missing or unauthenticated when PR fallback is needed, instruct the user to run `gh auth login` and re-run the skill; do not silently fail.

## Repo-context interface (example)

In `CLAUDE.md` or `AGENTS.md`:

```markdown
## Versioning

- **Base branch**: main
- **Commit message**: chore: bump version to {version}
- **Extra version files**: docs/install.md
- **Changelog categories**: Added, Improved, Fixed
```
````

- [ ] **Step 1.3: Add `## Versioning` to `AGENTS.md`**

Open `/Users/ether/workspace/personal/skill-set/AGENTS.md`. Append at the end (or in a logical location near other operational sections):

```markdown
## Versioning

- **Base branch**: main
- **Commit message**: chore: bump version to {version}
- **Extra version files**: (none)
- **Changelog categories**: Added, Improved, Fixed
```

- [ ] **Step 1.4: Create `.claude/settings.json` for hub dogfooding**

```bash
mkdir -p .claude
```

Write `/Users/ether/workspace/personal/skill-set/.claude/settings.json` with this content:

```json
{
  "extraKnownMarketplaces": {
    "ether-plugins": {
      "source": {
        "source": "github",
        "repo": "ether-moon/ether-plugins"
      }
    }
  },
  "enabledPlugins": {
    "skill-set@ether-plugins": true
  }
}
```

(`settings.local.json` retains permissions; the new `settings.json` only handles marketplace registration.)

- [ ] **Step 1.5: Commit**

```bash
git add plugins/skill-set/skills/bumping-version/SKILL.md AGENTS.md .claude/settings.json
git commit -m "feat: add generalized bumping-version skill and hub dogfooding"
```

- [ ] **Step 1.6: Push and create PR**

```bash
git push -u origin bumping-version-skill
gh pr create \
  --base main \
  --head bumping-version-skill \
  --title "feat: generalized bumping-version skill + hub dogfooding" \
  --body "$(cat <<'EOF'
## Summary

- Adds `bumping-version` skill at `plugins/skill-set/skills/bumping-version/SKILL.md`. Convention-based version-file detection (plugin.json, package.json, pyproject.toml, Cargo.toml, gemspec, VERSION, ...). Always works against the latest base branch via a temporary git worktree. Pushes to base; falls back to PR when base is protected.
- Adds `## Versioning` section to `AGENTS.md` so the new skill picks up self-policy.
- Adds `.claude/settings.json` registering the `ether-plugins` hub for in-repo dogfooding.

## Migration context

This is Phase 1 of the marketplace-refs unification spec at `ether-moon/ether-plugins:docs/superpowers/specs/2026-04-27-marketplace-refs-and-bumping-version-design.md`. After merge, the duplicated `bumping-version` skills in `knowledge-distillery` and `hotwire-frontend-skills` will be removed (Phase 2).

## Test plan

- [ ] Lint: SKILL.md frontmatter parses cleanly.
- [ ] Self-test: invoke the new skill on this repo to bump skill-set itself once Phase 1 is merged.
EOF
)"
```

Emit the PR URL.

- [ ] **Step 1.7: Restore working state**

```bash
git checkout main          # leave the dev environment on the default branch
```

---

## Task 2: Phase 2a — `knowledge-distillery` repo

**Wait for Task 1 PR to merge before starting.**

**Files:**
- Modify: `/Users/ether/workspace/personal/knowledge-distillery/.claude/settings.json`
- Modify: `/Users/ether/workspace/personal/knowledge-distillery/AGENTS.md` (add `## Versioning`)
- Delete: `/Users/ether/workspace/personal/knowledge-distillery/.claude/skills/bumping-version/`

**Branch:** `migrate-to-hub-marketplace`
**PR base:** `main`

- [ ] **Step 2.1: Sync and branch**

```bash
cd /Users/ether/workspace/personal/knowledge-distillery
git status                          # expect clean
git fetch origin
git checkout -b migrate-to-hub-marketplace origin/main
```

- [ ] **Step 2.2: Update `.claude/settings.json`**

Read the existing file first to preserve `permissions.allow` exactly. Replace the whole file with:

```json
{
  "permissions": {
    "allow": [
      "Bash(*/knowledge-gate:*)",
      "Bash(plugins/knowledge-distillery/scripts/knowledge-gate:*)",
      "Bash(sqlite3:*)"
    ]
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

If the actual `permissions.allow` array differs from the listing above (later additions), preserve what is on disk — only swap `extraKnownMarketplaces` and `enabledPlugins`.

- [ ] **Step 2.3: Add `## Versioning` to `AGENTS.md`**

Append to `/Users/ether/workspace/personal/knowledge-distillery/AGENTS.md`:

```markdown
## Versioning

- **Base branch**: main
- **Commit message**: chore: bump version to {version}
- **Extra version files**: (none — `plugins/knowledge-distillery/.claude-plugin/plugin.json` is auto-detected)
- **Changelog categories**: Added, Improved, Fixed
```

- [ ] **Step 2.4: Delete the old `bumping-version` skill**

```bash
git rm -r .claude/skills/bumping-version
```

If `.claude/skills/` becomes empty after removal, also remove the empty directory:

```bash
[ -d .claude/skills ] && [ -z "$(ls -A .claude/skills)" ] && rmdir .claude/skills
```

- [ ] **Step 2.5: Commit**

```bash
git add .claude/settings.json AGENTS.md
git status                          # confirm: settings.json modified, AGENTS.md modified, .claude/skills/bumping-version/ deleted
git commit -m "refactor: switch dogfooding to ether-plugins hub, drop local bumping-version skill"
```

- [ ] **Step 2.6: Push and create PR**

```bash
git push -u origin migrate-to-hub-marketplace
gh pr create \
  --base main \
  --head migrate-to-hub-marketplace \
  --title "refactor: switch dogfooding to ether-plugins hub" \
  --body "$(cat <<'EOF'
## Summary

- Replaces self-referential `knowledge-distillery@knowledge-distillery` dogfooding with `knowledge-distillery@ether-plugins`.
- Enables `skill-set@ether-plugins` so the generalized `bumping-version` skill is available in this repo.
- Removes the duplicated local `bumping-version` skill (now in `skill-set`).
- Adds `## Versioning` section to `AGENTS.md` declaring this repo's bump policy for the new skill to read.

## Migration context

Phase 2 of the marketplace-refs unification spec. Depends on Phase 1 (`skill-set` PR) being merged so the `bumping-version` skill is available.

## Test plan

- [ ] After merging, run `claude` in the repo and verify `bumping-version` is discoverable.
- [ ] Trigger a no-op dry-run of the bump (review only — do not push).
EOF
)"
```

- [ ] **Step 2.7: Restore working state**

```bash
git checkout main
```

---

## Task 3: Phase 2b — `hotwire-frontend-skills` (`hotwire-skills-integration` branch)

**Wait for Task 1 PR to merge before starting. Independent of Task 2.**

**Files (on `hotwire-skills-integration` branch):**
- Modify: `/Users/ether/workspace/personal/hotwire-frontend-skills/.claude/settings.json`
- Modify: `/Users/ether/workspace/personal/hotwire-frontend-skills/AGENTS.md`
- Delete: `/Users/ether/workspace/personal/hotwire-frontend-skills/.claude/skills/bumping-version/`

**Branch:** `migrate-to-hub-marketplace` (forked from `hotwire-skills-integration`)
**PR base:** `hotwire-skills-integration`

- [ ] **Step 3.1: Sync and branch**

```bash
cd /Users/ether/workspace/personal/hotwire-frontend-skills
git status                          # expect clean
git fetch origin
git checkout -b migrate-to-hub-marketplace hotwire-skills-integration
```

- [ ] **Step 3.2: Update `.claude/settings.json`**

Replace the file with:

```json
{
  "permissions": {
    "allow": []
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
    "hotwire-frontend-skills@ether-plugins": true,
    "skill-set@ether-plugins": true
  }
}
```

If `permissions.allow` on disk has entries, preserve them.

- [ ] **Step 3.3: Add `## Versioning` to `AGENTS.md`**

Append to `AGENTS.md`:

```markdown
## Versioning

- **Base branch**: hotwire-skills-integration
- **Commit message**: chore: bump version to {version}
- **Extra version files**: (none — `plugins/hotwire-frontend-skills/.claude-plugin/plugin.json` is auto-detected)
- **Changelog categories**: Added, Improved, Fixed
```

(Base branch is the integration branch since that's where this repo's plugin work lives until it's promoted to main.)

- [ ] **Step 3.4: Delete the old `bumping-version` skill**

```bash
git rm -r .claude/skills/bumping-version
[ -d .claude/skills ] && [ -z "$(ls -A .claude/skills)" ] && rmdir .claude/skills
```

- [ ] **Step 3.5: Commit**

```bash
git add .claude/settings.json AGENTS.md
git commit -m "refactor: switch dogfooding to ether-plugins hub, drop local bumping-version skill"
```

- [ ] **Step 3.6: Push and create PR**

```bash
git push -u origin migrate-to-hub-marketplace
gh pr create \
  --base hotwire-skills-integration \
  --head migrate-to-hub-marketplace \
  --title "refactor: switch dogfooding to ether-plugins hub" \
  --body "$(cat <<'EOF'
## Summary

- Replaces self-referential `hotwire-frontend-skills@hotwire-frontend-skills` dogfooding with `hotwire-frontend-skills@ether-plugins`.
- Enables `skill-set@ether-plugins` so the generalized `bumping-version` skill is available.
- Removes the duplicated local `bumping-version` skill (now in `skill-set`); its description was also stale ("cc-plugin-template").
- Adds `## Versioning` section to `AGENTS.md` declaring this repo's bump policy.

## Migration context

Phase 2 of the marketplace-refs unification spec. Depends on Phase 1 (`skill-set` PR) being merged.

## Test plan

- [ ] After merging, run `claude` in the repo and verify `bumping-version` is discoverable.
EOF
)"
```

- [ ] **Step 3.7: Restore working state**

```bash
git checkout hotwire-skills-integration
```

---

## Task 4: Phase 3 — `cc-plugin-template` (`hotwire-frontend-skills/main` branch)

**Independent of Tasks 1–3. Can run in parallel.**

**Files (on `main` branch of `hotwire-frontend-skills`):**
- Modify: `/Users/ether/workspace/personal/hotwire-frontend-skills/init.sh`
- Modify: `/Users/ether/workspace/personal/hotwire-frontend-skills/templates/project/settings.json`
- Modify: `/Users/ether/workspace/personal/hotwire-frontend-skills/templates/project/AGENTS.md`

**Branch:** `template-hub-default`
**PR base:** `main`

- [ ] **Step 4.1: Sync and branch**

```bash
cd /Users/ether/workspace/personal/hotwire-frontend-skills
git status
git fetch origin
git checkout -b template-hub-default origin/main
```

- [ ] **Step 4.2: Edit `init.sh` — add hub prompt and substitution**

Locate the existing `read -rp "GitHub owner ..."` line and add the hub prompt right after it. The simplest design is "always render hub-style; if user gives empty hub, default it to the per-plugin marketplace name so generation is identical to today."

Replace this block:

```bash
read -rp "GitHub owner (user or org): " GITHUB_OWNER

# Defaults
YEAR=$(date +%Y)
```

with:

```bash
read -rp "GitHub owner (user or org): " GITHUB_OWNER
read -rp "Hub marketplace name (kebab-case, blank = use this plugin as its own marketplace): " HUB_MARKETPLACE_NAME
read -rp "Hub GitHub owner (blank = same as plugin owner): " HUB_GITHUB_OWNER

# Default hub to per-plugin self-registration when blank
HUB_MARKETPLACE_NAME="${HUB_MARKETPLACE_NAME:-$MARKETPLACE_NAME}"
HUB_GITHUB_OWNER="${HUB_GITHUB_OWNER:-$GITHUB_OWNER}"

# Defaults
YEAR=$(date +%Y)
```

In the `render()` function, add two `-e` lines for the new substitutions. Locate:

```bash
sed \
    -e "s|{{MARKETPLACE_NAME}}|${MARKETPLACE_NAME}|g" \
    -e "s|{{PLUGIN_NAME}}|${PLUGIN_NAME}|g" \
    -e "s|{{PLUGIN_DESCRIPTION}}|${PLUGIN_DESCRIPTION}|g" \
    -e "s|{{AUTHOR_NAME}}|${AUTHOR_NAME}|g" \
    -e "s|{{GITHUB_OWNER}}|${GITHUB_OWNER}|g" \
    -e "s|{{YEAR}}|${YEAR}|g" \
    -e "s|{{MONTH}}|${MONTH}|g" \
    -e "s|{{DAY}}|${DAY}|g" \
    "$src" > "$dst"
```

Replace with:

```bash
sed \
    -e "s|{{MARKETPLACE_NAME}}|${MARKETPLACE_NAME}|g" \
    -e "s|{{HUB_MARKETPLACE_NAME}}|${HUB_MARKETPLACE_NAME}|g" \
    -e "s|{{HUB_GITHUB_OWNER}}|${HUB_GITHUB_OWNER}|g" \
    -e "s|{{PLUGIN_NAME}}|${PLUGIN_NAME}|g" \
    -e "s|{{PLUGIN_DESCRIPTION}}|${PLUGIN_DESCRIPTION}|g" \
    -e "s|{{AUTHOR_NAME}}|${AUTHOR_NAME}|g" \
    -e "s|{{GITHUB_OWNER}}|${GITHUB_OWNER}|g" \
    -e "s|{{YEAR}}|${YEAR}|g" \
    -e "s|{{MONTH}}|${MONTH}|g" \
    -e "s|{{DAY}}|${DAY}|g" \
    "$src" > "$dst"
```

- [ ] **Step 4.3: Update `templates/project/settings.json`**

Replace the file content with:

```json
{
  "permissions": {
    "allow": []
  },
  "extraKnownMarketplaces": {
    "{{HUB_MARKETPLACE_NAME}}": {
      "source": {
        "source": "github",
        "repo": "{{HUB_GITHUB_OWNER}}/{{HUB_MARKETPLACE_NAME}}"
      }
    }
  },
  "enabledPlugins": {
    "{{PLUGIN_NAME}}@{{HUB_MARKETPLACE_NAME}}": true,
    "skill-set@{{HUB_MARKETPLACE_NAME}}": true
  }
}
```

When the user gives a blank hub, init.sh will default `HUB_MARKETPLACE_NAME=$MARKETPLACE_NAME` and `HUB_GITHUB_OWNER=$GITHUB_OWNER`, producing settings identical to today (per-plugin self-registration), with the addition of `skill-set@<that-marketplace>` enabled. If `skill-set` is not in that marketplace, it simply won't load — harmless.

- [ ] **Step 4.4: Add `## Versioning` placeholder to `templates/project/AGENTS.md`**

Append to `/Users/ether/workspace/personal/hotwire-frontend-skills/templates/project/AGENTS.md`:

```markdown
## Versioning

<!-- Filled in per project. The `bumping-version` skill (in skill-set) reads this section. All keys optional — defaults apply when omitted. -->

- **Base branch**:
- **Commit message**:
- **Extra version files**:
- **Changelog categories**:
```

- [ ] **Step 4.5: Smoke-test the template (optional but recommended)**

```bash
TMPDIR=$(mktemp -d)
cp -r init.sh templates "$TMPDIR/"
cd "$TMPDIR"
echo -e "test-mp\ntest-plug\nA test plugin\nTest Author\ntest-owner\nether-plugins\nether-moon" | bash init.sh
cat .claude/settings.json
cat AGENTS.md | grep -A 5 "## Versioning"
cd -
rm -rf "$TMPDIR"
```

Expected: `.claude/settings.json` references `ether-plugins` and `ether-moon/ether-plugins`; AGENTS.md contains the Versioning placeholder.

- [ ] **Step 4.6: Commit**

```bash
cd /Users/ether/workspace/personal/hotwire-frontend-skills
git add init.sh templates/project/settings.json templates/project/AGENTS.md
git commit -m "feat(template): default to hub marketplace and include skill-set + Versioning placeholder"
```

- [ ] **Step 4.7: Push and create PR**

```bash
git push -u origin template-hub-default
gh pr create \
  --base main \
  --head template-hub-default \
  --title "feat(template): hub marketplace as default + Versioning placeholder" \
  --body "$(cat <<'EOF'
## Summary

- `init.sh` adds two new prompts: hub marketplace name and hub GitHub owner. Both default to the per-plugin values when blank, so existing single-marketplace setups still work.
- `templates/project/settings.json` is rewritten in hub-style and additionally enables `skill-set@<hub>` so the generated plugin can use `bumping-version` and other skill-set tools out of the box.
- `templates/project/AGENTS.md` gets an empty `## Versioning` section the user fills in per project.

## Why

Phase 3 of the marketplace-refs unification spec. New plugins generated from this template now start out hub-aligned by default.

## Test plan

- [ ] Run `bash init.sh` against a temp directory with both blank and non-blank hub inputs; confirm generated settings.json reflects each.
EOF
)"
```

- [ ] **Step 4.8: Restore working state**

```bash
git checkout main
```

---

## Task 5: Phase 4 — `ether-plugins` hub README note

**Independent of all other tasks. Already on `fix-cross-plugin-refs` branch in this workspace.**

**Files:**
- Modify: `/Users/ether/conductor/workspaces/ether-plugins-v1/osaka-v2/README.md`

**Branch:** `fix-cross-plugin-refs` (current branch in this workspace)
**PR base:** `main`

- [ ] **Step 5.1: Edit README — add dogfooding note**

Locate the "How it works" section. After the existing JSON snippet showing the `git-subdir` source format, append a new subsection:

```markdown
### Dogfooding inside plugin repos

Each plugin's source repository registers this hub in its own `.claude/settings.json` for in-repo development:

```json
{
  "extraKnownMarketplaces": {
    "ether-plugins": {
      "source": { "source": "github", "repo": "ether-moon/ether-plugins" }
    }
  },
  "enabledPlugins": {
    "<plugin-name>@ether-plugins": true,
    "skill-set@ether-plugins": true
  }
}
```

This means a developer opening any plugin repo with Claude Code gets the plugin itself plus the shared `skill-set` tooling. A brand-new plugin must be added to this hub's `marketplace.json` before in-repo dogfooding can resolve.
```

- [ ] **Step 5.2: Commit**

```bash
cd /Users/ether/conductor/workspaces/ether-plugins-v1/osaka-v2
git add README.md
git commit -m "docs: explain hub dogfooding pattern used by plugin repos"
```

- [ ] **Step 5.3: Push and create PR**

```bash
git push -u origin fix-cross-plugin-refs
gh pr create \
  --base main \
  --head fix-cross-plugin-refs \
  --title "docs: explain hub dogfooding pattern + design + plan" \
  --body "$(cat <<'EOF'
## Summary

- Adds design and implementation plan for the marketplace-refs unification under `docs/superpowers/specs/` and `docs/superpowers/plans/`.
- Adds a "Dogfooding inside plugin repos" subsection to the README explaining how each plugin repo references this hub via its own `.claude/settings.json`.

## Sibling PRs

This is Phase 4 of a 4-phase rollout. The other phases ship in their own repos:

- skill-set: `bumping-version` skill + hub dogfooding
- knowledge-distillery: switch dogfooding to hub, drop local skill
- hotwire-frontend-skills (integration branch): same as above
- hotwire-frontend-skills (main / cc-plugin-template): hub-default template

## Test plan

- [ ] Render the README and verify the new subsection.
EOF
)"
```

---

## Self-review (post-write)

- **Spec coverage:** Goals 1–5 in the spec map to Tasks 1–5 (skill creation in 1, settings/AGENTS edits in 2 and 3, template alignment in 4, hub README in 5). Non-goal "remove per-plugin marketplace.json" honored (no such deletion). Out-of-scope repos (`agent-atelier`, `herb-lsp-plugin`) untouched.
- **Placeholder scan:** No `TBD` / `TODO` / vague handwaving in steps. Every code block is the literal content to write.
- **Type/path consistency:** Branch names match across push and `gh pr create --head` flags. PR base branches match each repo's default. Filesystem paths absolute and verified against earlier exploration. The `bumping-version` SKILL.md content matches the spec's workflow numbering 0–9.
- **Sequencing:** Task 1 must merge before Tasks 2 and 3 (consumers enable `skill-set@ether-plugins`). Tasks 4 and 5 are independent. Each task ends with a PR per the user's requirement.
