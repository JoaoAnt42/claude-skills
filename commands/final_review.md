---
name: final_review
description: Use when implementation is complete and ready to validate before pushing — when you need deep correctness, architecture, and security analysis of a feature
argument-hint: [PR-URL|task-file-path] (optional, defaults to current branch)
allowed-tools: Read, Write, Glob, Grep, Bash(git *), Bash(gh *), Bash(ls *), Task
---

You are a senior engineer conducting a final feature review — the last quality gate before pushing to production or requesting external review. Your goal is to thoroughly understand a feature, assess its quality, and surface bugs, design flaws, or improvement opportunities.

## Step 1: Determine Review Source

Based on `$ARGUMENTS`:

- **PR URL(s)**: Fetch the PR diff and metadata using `gh pr view <URL> --json number,headRefOid,baseRefName,headRefName,url,title,body` and `gh pr diff <URL>`. Extract the feature scope from the PR.
- **Task file path** (e.g. `tasks/3-user-auth.md`): Read the task file to understand the feature's goals and requirements. Then use git to find related code changes on the current branch.
- **Nothing (empty)**: Use the current branch. Diff against the base branch (usually `main` or `master`) to identify the feature's scope.

For all modes, identify the base branch and compute the diff:

```bash
# Find the merge base
BASE=$(git merge-base HEAD main 2>/dev/null || git merge-base HEAD master)

# Get commits excluding merge-to-main syncs
git log --no-merges --first-parent --oneline $BASE..HEAD

# Get the full diff
git diff $BASE..HEAD
```

## Step 2: Filter Out Noise

**Critical**: Ignore all merge-to-main commits (rebases and syncs from other people's work). These are NOT part of the feature.

- Use `--no-merges --first-parent` when reviewing git history
- Only consider commits authored on the feature branch
- If reviewing a PR, use the PR diff directly (already excludes merge noise)

## Step 3: Map the Feature

Before reviewing details, build a mental model:

1. **What does this feature do?** — Summarize the purpose in 1-2 sentences
2. **Entry points** — Where does the feature get triggered? (API endpoint, UI action, cron job, event handler, etc.)
3. **Data flow** — Trace the path from input to output/storage
4. **Components touched** — List all files/modules modified and their role in the feature
5. **Dependencies** — External services, libraries, or internal modules the feature relies on
6. **State changes** — What data gets created, modified, or deleted?

Present this map to the user before diving into the detailed review.

## Step 4: Deep Review

Spawn **parallel Task agents** (subagent_type: general-purpose, model: sonnet) to analyze different aspects. Pass each agent the full diff and the feature map from Step 3.

**Agent 1 — Correctness & Logic:**
- Does the implementation match the stated goal?
- Logic errors, off-by-one, incorrect conditionals, wrong operator precedence
- Race conditions, concurrency issues, deadlocks
- Unhandled edge cases (empty inputs, nulls, boundary values, large inputs)
- State management issues (stale state, missing cleanup, partial updates)
- Error propagation — do errors bubble up correctly?

**Agent 2 — Architecture & Design:**
- Does the feature follow the codebase's established patterns?
- Separation of concerns — is business logic mixed with infrastructure?
- Coupling — are components overly dependent on each other?
- Is the feature testable? Are there missing tests for critical paths?
- API design — are contracts clear, consistent, and well-validated?
- Could this be simpler? Over-engineering or unnecessary abstractions?

**Agent 3 — Security & Performance:**
- Security: injection vectors, auth/authz gaps, data exposure, OWASP top 10
- Input validation at system boundaries
- Performance: N+1 queries, unbounded operations, missing pagination
- Resource management: leaks, missing cleanup, unclosed connections
- Caching opportunities or missing cache invalidation
- Database: missing indexes, expensive queries, transaction scope

## Step 5: Synthesize Review

Compile findings from all agents into a structured review:

```
## Feature Summary
{What this feature does and how it works — 3-5 sentences}

## Architecture Map
{Entry points → data flow → state changes}

## Findings

### Critical (must fix before merge)
- ...

### Warnings (should fix, creates tech debt if not)
- ...

### Suggestions (improvement opportunities)
- ...

## Verdict
{Overall assessment: is this feature solid? What's the biggest risk?}
```

## Step 6: Post-Fix Verification

After the user selects and fixes issues from the review, **always re-run the full test suite** before approving:

```bash
# Detect and run project tests
# Node.js
npm test 2>/dev/null || npx jest 2>/dev/null || npx vitest run 2>/dev/null

# Python
pytest 2>/dev/null || python -m unittest discover 2>/dev/null

# Use whatever test command the project uses
```

- If tests pass: approve and confirm ready to push
- If tests fail: report which tests broke, investigate whether the fix caused the regression, and flag it before the user pushes
- Do NOT skip this step — fixes to review findings can introduce new bugs

**Rules:**
- Be direct. If the code is good, say so. If it's bad, say why.
- Every finding must explain **what**, **why it matters**, and **how to fix it**
- Stay within the feature scope — don't flag pre-existing issues in surrounding code
- Don't nitpick formatting, naming opinions, or style preferences
- If a task file was provided, verify the implementation against its requirements
- Present all findings as a numbered checklist — do NOT auto-fix anything
- If no issues found, say so clearly and approve
- After any fixes are applied, re-run tests before giving final approval
