---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# Requesting Code Review

Dispatch a code reviewer subagent to catch issues before they cascade. The reviewer gets precisely crafted context for evaluation — never your session's history. This keeps the reviewer focused on the work product, not your thought process, and preserves your own context for continued work.

**Core principle:** Review early, review often.

## When to Request Review

**Mandatory:**
- After each task in subagent-driven development
- After completing major feature
- Before merge to main

**Optional but valuable:**
- When stuck (fresh perspective)
- Before refactoring (baseline check)
- After fixing complex bug

## How to Request

**1. Get the revision range** — select by intent, not by fixed revsets:

- `HEAD` = the commit containing the finished work. That is `@` if you work in it,
  but `@-` when `@` is an empty working-copy change on top of the finished commit
  (the common jj state). Check: `jj log -r @ --no-graph -T 'empty'` prints `true` → use `@-`.
- `BASE` = the commit you branched from (a base bookmark like `main`, or the fork
  point `heads(::@ & ::main)`) — NOT simply the parent of HEAD, which drops all but
  the last commit of multi-commit work.

```bash
BASE_SHA=$(jj log -r 'main' --no-graph -T commit_id)   # or the fork point revset above
HEAD_SHA=$(jj log -r '@-'  --no-graph -T commit_id)    # use @ if your work lives in @
```

**2. Dispatch code reviewer subagent:**

Dispatch a `general-purpose` subagent, filling the template at [code-reviewer.md](code-reviewer.md)

**Placeholders:**
- `[DESCRIPTION]` - Brief summary of what you built
- `[PLAN_OR_REQUIREMENTS]` - What it should do
- `[BASE_SHA]` - Starting commit
- `[HEAD_SHA]` - Ending commit

Only these four are dispatcher-filled. Other bracketed text inside the template (e.g. the Output Format section) is guidance for the reviewer — leave it verbatim.

**3. Act on feedback:**
- Fix Critical issues immediately
- Fix Important issues before proceeding
- Note Minor issues for later
- Push back if reviewer is wrong (with reasoning)

## Example

```
[Just completed Task 2: Add verification function]

You: Let me request code review before proceeding.

$ jj log -r @ --no-graph -T 'empty'
true    # @ is an empty working-copy change → the finished work is @-

BASE_SHA=$(jj log -r 'description("Task 1")' --no-graph -T 'commit_id ++ "\n"' | head -1)
HEAD_SHA=$(jj log -r '@-' --no-graph -T commit_id)

[Dispatch code reviewer subagent]
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[Subagent returns]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

You: [Fix progress indicators]
[Continue to Task 3]
```

## Integration with Workflows

**Subagent-Driven Development:**
- Review after EACH task
- Catch issues before they compound
- Fix before moving to next task

**Executing Plans:**
- Review after each task or at natural checkpoints
- Get feedback, apply, continue

**Ad-Hoc Development:**
- Review before merge
- Review when stuck

## Red Flags

**Never:**
- Skip review because "it's simple"
- Ignore Critical issues
- Proceed with unfixed Important issues
- Argue with valid technical feedback

**If reviewer wrong:**
- Push back with technical reasoning
- Show code/tests that prove it works
- Request clarification

See template at: [code-reviewer.md](code-reviewer.md)
