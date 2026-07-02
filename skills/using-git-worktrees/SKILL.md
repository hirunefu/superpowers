---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - ensures an isolated workspace exists via native tools or git worktree fallback
---

# Using Git Worktrees

## Overview

Ensure work happens in an isolated workspace. Prefer your platform's native worktree tools. Fall back to manual git worktrees only when no native tool is available.

**Core principle:** Detect existing isolation first. Then use native tools. Then fall back to git. Never fight the harness.

**Version control:** This fork drives commits, bookmarks, and merges with `jj` (Jujutsu). Isolation is jj-first: in a jj-colocated repo, create additional workspaces with `jj workspace add`. Colocating jj inside a `git worktree` is impossible (`jj git init --colocate` refuses with "Cannot create a colocated jj repo inside a Git worktree", jj 0.42), and jj commands run inside one silently target the main checkout instead. Git worktrees remain only for harness-owned isolation and no-jj fallback — operate those with git.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Step 0: Detect Existing Isolation

**Before creating anything, check if you are already in an isolated workspace.**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
GIT_TOPLEVEL=$(git rev-parse --show-toplevel 2>/dev/null)
JJ_ROOT=$(jj root 2>/dev/null)
BRANCH=$(git branch --show-current)
```

**Submodule guard:** `GIT_DIR != GIT_COMMON` is also true inside git submodules. Before concluding "already in a worktree," verify you are not in a submodule:

```bash
# If this returns a path, you're in a submodule, not a worktree — treat as normal repo
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**If `GIT_DIR != GIT_COMMON` (and not a submodule):** You are already in a linked git worktree. Do NOT create another workspace. Operate this worktree with **git** — jj cannot manage it (see Step 2 caveat). Skip to Step 3.

Report with branch state:
- On a branch: "Already in isolated workspace at `<path>` on branch `<name>`."
- Detached HEAD: "Already in isolated workspace at `<path>` (detached HEAD, externally managed). Branch creation needed at finish time."

**If `JJ_ROOT` is non-empty and differs from `GIT_TOPLEVEL`:** You are already inside a secondary jj workspace (its root is `JJ_ROOT`) — isolated and jj-managed. Do NOT create another workspace. Skip to Step 3.

**If `GIT_DIR == GIT_COMMON` (or in a submodule):** You are in a normal repo checkout.

Has the user already indicated their worktree preference in your instructions? If not, ask for consent before creating a worktree:

> "Would you like me to set up an isolated worktree? It protects your current branch from changes."

Honor any existing declared preference without asking. If the user declines consent, work in place and skip to Step 2.

## Step 1: Create Isolated Workspace

**You have two mechanisms. Try them in this order.**

### 1a. Native Worktree Tools (preferred)

The user has asked for an isolated workspace (Step 0 consent). Do you already have a way to create a worktree? It might be a tool with a name like `EnterWorktree`, `WorktreeCreate`, a `/worktree` command, or a `--worktree` flag. Use it only if it operates on the repository you are working in — a harness tool that manages some other project directory does not apply. If it applies, use it and skip to Step 2.

Native tools handle directory placement, branch creation, and cleanup automatically. Using `git worktree add` when you have a native tool creates phantom state your harness can't see or manage.

**jj caveat:** a native tool creates a git worktree, and jj cannot colocate inside one. Operate a harness-created worktree with **git**; jj resumes at the main checkout at integration time.

Only proceed to Step 1b if you have no applicable native worktree tool.

### 1b. Workspace Fallback (jj workspace, or git worktree without jj)

**Only use this if Step 1a does not apply** — you have no native worktree tool available. Create the isolation manually: with `jj workspace add` when jj is available (this fork's default), with `git worktree add` only when it is not.

First bind the workspace name — later blocks depend on it:

```bash
BRANCH_NAME=<the feature/workspace name you are about to create, e.g. add-pow>
```

#### Directory Selection

Follow this priority order. Explicit user preference always beats observed filesystem state.

1. **Check your instructions for a declared worktree directory preference.** If the user has already specified one, use it without asking.

2. **Check for an existing project-local worktree directory:**
   ```bash
   ls -d .worktrees 2>/dev/null     # Preferred (hidden)
   ls -d worktrees 2>/dev/null      # Alternative
   ```
   If found, use it. If both exist, `.worktrees` wins.

3. **If there is no other guidance available**, default to `.worktrees/` at the project root.

#### Safety Verification (project-local directories only)

**MUST verify directory is ignored before creating worktree:**

```bash
# Check the actual path you are about to create, NOT the bare directory name.
# `git check-ignore -q .worktrees` returns a false negative when the directory
# does not exist yet: a trailing-slash rule like `.worktrees/` in .gitignore
# matches paths under it, but the bare token won't match a not-yet-created dir.
WORKTREE_PARENT=.worktrees   # or `worktrees`, per the directory selection above
git check-ignore -q "$WORKTREE_PARENT/$BRANCH_NAME"
```

**If NOT ignored:** Add the worktree parent directory to .gitignore, commit the change, then proceed. Use **git** for this commit (`git add .gitignore && git commit`) — it advances the checked-out branch, so the rule lands directly in `main`'s history. (`jj commit` also works in a colocated repo, but it leaves the `main` bookmark behind on the parent, which surprises later steps.) The commit lands on the original checkout before the workspace exists.

**Why critical:** Prevents workspace contents from being tracked twice over — git would commit them, and the main checkout's jj working copy would sweep them into its snapshots (jj respects .gitignore, so an ignored parent directory protects both).

#### Create the Workspace

```bash
# NOTE: if your shell does not persist state between commands, re-bind first:
# GIT_TOPLEVEL from Step 0, WORKTREE_PARENT and BRANCH_NAME from Step 1b above.

# jj-first: ensure the MAIN checkout is colocated, then create a jj workspace.
# Colocation must happen here, BEFORE creating isolation — `jj git init --colocate`
# inside a git worktree is impossible (jj refuses).
[ -d "$GIT_TOPLEVEL/.jj" ] || jj git init --colocate   # run at the main checkout root

# Same WORKTREE_PARENT the Safety Verification above checked.
# jj workspace add does NOT create missing parent directories — make it first.
mkdir -p "$WORKTREE_PARENT"
path="$WORKTREE_PARENT/$BRANCH_NAME"
jj workspace add "$path"
cd "$path"
```

No upfront branch is needed: jj tracks the work as changes, and the bookmark is
created at finish time (`finishing-a-development-branch` names the work when
integrating or pushing).

**No jj available:** fall back to `git worktree add "$path" -b "$BRANCH_NAME"` and
operate that worktree with git.

**Sandbox fallback:** If workspace creation fails with a permission error (sandbox denial), tell the user the sandbox blocked it and you're working in the current directory instead. Then run setup and baseline tests in place.

## Step 2: Verify jj Owns the Workspace

The workspace exists (native tool, jj workspace, git worktree, or in-place). Confirm
which VCS actually manages it before doing any work:

```bash
# A jj-owned workspace has its own .jj and jj agrees on the root:
[ "$(jj root 2>/dev/null)" = "$(pwd -P)" ] && echo "jj-owned" || echo "NOT jj-owned"
```

Do not use `jj st` as the ownership probe — it succeeds by discovering an ancestor
repo's `.jj` and reports the *main checkout's* state, a false positive exactly when
you are inside a fresh worktree.

- **jj-owned** (jj workspace, or in-place colocated checkout): use jj exclusively here.
  Caveat: git commands inside a jj workspace resolve to the MAIN repo — `git rev-parse
  HEAD` reports the main checkout, not your workspace. Don't mix.
- **NOT jj-owned** (harness-created or fallback git worktree): use git exclusively here.
  jj commands would silently target the main checkout.
- **In-place work in a plain git checkout** (user declined isolation): colocate now —
  `[ -d .jj ] || jj git init --colocate` (allowed here — this is the main checkout).

From here on, this skill family uses jj for version control where jj owns the
workspace (see `finishing-a-development-branch` for commit/merge/PR flow).

## Step 3: Project Setup

Auto-detect and run appropriate setup:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## Step 4: Verify Clean Baseline

Run tests to ensure workspace starts clean:

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### Report

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Already in linked git worktree | Skip creation; operate with git (Step 0) |
| Already in a jj workspace | Skip creation; jj-managed (Step 0) |
| In a submodule | Treat as normal repo (Step 0 guard) |
| Native worktree tool available (for this repo) | Use it; operate with git (Step 1a) |
| No native tool, jj available | `jj workspace add` fallback (Step 1b) |
| No native tool, no jj | `git worktree add` fallback (Step 1b) |
| `.worktrees/` exists | Use it (verify ignored) |
| `worktrees/` exists | Use it (verify ignored) |
| Both exist | Use `.worktrees/` |
| Neither exists | Check instruction file, then default `.worktrees/` |
| Directory not ignored | Add to .gitignore + commit |
| Workspace ready (created or in place) | Verify jj ownership (Step 2) |
| Permission error on create | Sandbox fallback, work in place |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## Common Mistakes

### Fighting the harness

- **Problem:** Using `git worktree add` when the platform already provides isolation
- **Fix:** Step 0 detects existing isolation. Step 1a defers to native tools.

### Skipping detection

- **Problem:** Creating a nested worktree inside an existing one
- **Fix:** Always run Step 0 before creating anything

### Skipping ignore verification

- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always use `git check-ignore` before creating project-local worktree

### Assuming directory location

- **Problem:** Creates inconsistency, violates project conventions
- **Fix:** Follow priority: explicit instructions > existing project-local directory > default

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

### Probing jj ownership with `jj st`

- **Problem:** `jj st` succeeds by discovering an ancestor repo's `.jj` — inside a fresh git worktree it reports the main checkout's state and makes the worktree look jj-managed
- **Fix:** Probe with `[ "$(jj root)" = "$(pwd -P)" ]` (Step 2), which checks the exact workspace you are in

## Red Flags

**Never:**
- Create a workspace when Step 0 detects existing isolation
- Use manual fallback when you have an applicable native worktree tool (e.g., `EnterWorktree`). This is the #1 mistake — if you have it, use it.
- Skip Step 1a by jumping straight to Step 1b's commands
- Create a workspace without verifying its parent directory is ignored (project-local)
- Run `jj git init --colocate` inside a git worktree (jj refuses; and jj commands there silently target the main checkout)
- Mix VCSs in one workspace: jj workspace → jj only; git worktree → git only
- Skip baseline test verification
- Proceed with failing tests without asking

**Always:**
- Run Step 0 detection first
- Prefer native tools over manual fallback
- Follow directory priority: explicit instructions > existing project-local directory > default
- Verify directory is ignored for project-local
- Verify which VCS owns the workspace (Step 2) before any commit/bookmark/merge work
- Auto-detect and run project setup
- Verify clean test baseline
