---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# Finishing a Development Branch

## Overview

Guide completion of development work by presenting clear options and handling chosen workflow.

**Core principle:** Verify tests → Detect environment → Present options → Execute choice → Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Detect Environment

**Determine workspace state before presenting options:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

This determines which menu to show and how cleanup works:

| State | Menu | Cleanup |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON` (normal repo) | Standard 4 options | No worktree to clean up |
| `GIT_DIR != GIT_COMMON` (linked worktree) | Standard 4 options | Provenance-based (see Step 6) |

jj has no "detached HEAD" state: the working change (`@`) always exists, and any change can
be given a bookmark at push time. So there is no reduced menu — always present the 4 options.
When the workspace is externally managed (harness-owned), Step 6's provenance check still
prevents you from removing a worktree you didn't create.

### Step 3: Determine Base Bookmark

```bash
# Fork point from the base bookmark (usually main, else master)
jj log -r 'heads(::@ & ::main)' --no-graph -T 'commit_id ++ "\n"' 2>/dev/null \
  || jj log -r 'heads(::@ & ::master)' --no-graph -T 'commit_id ++ "\n"' 2>/dev/null
```

Or ask: "This work split from main - is that correct?"

### Step 4: Present Options

**Present exactly these 4 options:**

```
Implementation complete. What would you like to do?

1. Merge back to <base-bookmark> locally
2. Push and create a Pull Request
3. Keep the work as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Don't add explanation** - keep options concise.

### Step 5: Execute Choice

#### Option 1: Merge Locally

**Solo flow only.** This fast-forwards the base bookmark locally. For a shared base (team
work), use Option 2 (PR) instead — don't advance a shared base by hand.

```bash
# Get main repo root for CWD safety
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# Integrate locally. ONLY for changes you have NOT pushed/shared —
# rebasing rewrites commit IDs and would diverge from anything already published.
jj git fetch                            # refresh the base bookmark
jj rebase -d <base-bookmark>            # replay your work on the latest base
jj bookmark set <base-bookmark> -r @    # fast-forward base to include it

# Verify tests on the integrated result
<test command>
```

Pushing the advanced base is a separate, deliberate step (`jj git push --bookmark <base-bookmark>`).

Then: Cleanup worktree (Step 6), then delete the feature bookmark if you created one:

```bash
jj bookmark delete <feature-bookmark>   # only if one exists
```

#### Option 2: Push and Create PR

This is the team integration path: push a bookmark and let the host's merge policy
(squash/merge/rebase, configured on the repo) apply when the PR merges.

```bash
# Name the work if it has no bookmark yet
jj bookmark create <feature-bookmark> -r @   # skip if it already has one

# Push the bookmark to the remote
jj git push --bookmark <feature-bookmark>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

**Do NOT clean up worktree** — user needs it alive to iterate on PR feedback.

#### Option 3: Keep As-Is

Report: "Keeping work on bookmark <name>. Worktree preserved at <path>."

**Don't cleanup worktree.**

#### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Bookmark <name> (if any)
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation.

If confirmed:
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

Then: Cleanup worktree (Step 6), then abandon the work:
```bash
jj abandon <feature-change>             # drop the commits
jj bookmark delete <feature-bookmark>   # only if one exists
```

### Step 6: Cleanup Workspace

**Only runs for Options 1 and 4.** Options 2 and 3 always preserve the worktree.

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**If `GIT_DIR == GIT_COMMON`:** Normal repo, no worktree to clean up. Done.

**If worktree path is under `.worktrees/`, `worktrees/`, or `~/.config/superpowers/worktrees/`:** Superpowers created this worktree — we own cleanup.

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # Self-healing: clean up any stale registrations
```

**Otherwise:** The host environment (harness) owns this workspace. Do NOT remove it. If your platform provides a workspace-exit tool, use it. Otherwise, leave the workspace in place.

## Quick Reference

| Option | Integrate | Push | Keep Worktree | Cleanup Bookmark |
|--------|-----------|------|---------------|------------------|
| 1. Merge locally | fast-forward base | - | - | delete (if any) |
| 2. Create PR | - | yes | yes | - |
| 3. Keep as-is | - | - | yes | - |
| 4. Discard | - | - | - | abandon + delete |

## Common Mistakes

**Skipping test verification**
- **Problem:** Merge broken code, create failing PR
- **Fix:** Always verify tests before offering options

**Open-ended questions**
- **Problem:** "What should I do next?" is ambiguous
- **Fix:** Present exactly 4 structured options (or 3 for detached HEAD)

**Cleaning up worktree for Option 2**
- **Problem:** Remove worktree user needs for PR iteration
- **Fix:** Only cleanup for Options 1 and 4

**Deleting the bookmark before integrating**
- **Problem:** Abandoning or deleting the work before it's merged or pushed loses it
- **Fix:** Integrate (Option 1) or push (Option 2) first, then delete the bookmark

**Running git worktree remove from inside the worktree**
- **Problem:** Command fails silently when CWD is inside the worktree being removed
- **Fix:** Always `cd` to main repo root before `git worktree remove`

**Cleaning up harness-owned worktrees**
- **Problem:** Removing a worktree the harness created causes phantom state
- **Fix:** Only clean up worktrees under `.worktrees/`, `worktrees/`, or `~/.config/superpowers/worktrees/`

**No confirmation for discard**
- **Problem:** Accidentally delete work
- **Fix:** Require typed "discard" confirmation

## Red Flags

**Never:**
- Proceed with failing tests
- Integrate without verifying tests on result
- Delete work without confirmation
- Rebase changes that have already been pushed or shared (rewrites published commits)
- Push a bookmark non-fast-forward (rewriting remote history) without explicit request
- Remove a worktree before confirming integration success
- Clean up worktrees you didn't create (provenance check)
- Run `git worktree remove` from inside the worktree

**Always:**
- Verify tests before offering options
- Detect environment before presenting menu
- Present exactly 4 options (or 3 for detached HEAD)
- Get typed confirmation for Option 4
- Clean up worktree for Options 1 & 4 only
- `cd` to main repo root before worktree removal
- Run `git worktree prune` after removal
