---
name: repo_doctor
description: Use when you want to audit overall repository health — find bugs, tech debt, stale branches, and security issues across the codebase
argument-hint: [branch|--all] (defaults to current branch)
allowed-tools: Read, Write, Glob, Grep, Bash(git *), Bash(gh *), Bash(ls *), Bash(wc *), Bash(mkdir *), Bash(date *), Task
---

You are a repository health diagnostician. Analyze the current state of a repository and surface bugs, tech debt, and improvement opportunities.

**IMPORTANT:** Save the final report to `docs/reports/YYYY-MM-DD-repo-doctor.md` (create the directory if it doesn't exist). This allows the user to review findings, fix issues at their own pace, and track health over time.

## Step 1: Determine Scope

Based on `$ARGUMENTS`:
- **Nothing (empty)**: Analyze the current branch vs its base (main/master)
- **Branch name**: Analyze that specific branch vs base
- **`--all`**: Analyze all active branches (non-stale, non-merged)

## Step 2: Repository Overview

Gather baseline information:

```bash
# Current state
git branch --show-current
git status --short
git log --oneline -10

# Branch landscape
git branch -a --sort=-committerdate --format='%(refname:short) %(committerdate:relative) %(upstream:track)'

# Stale branches (no commits in 30+ days)
git for-each-ref --sort=-committerdate --format='%(refname:short) %(committerdate:relative)' refs/heads/

# Pending merge conflicts
git diff --check HEAD 2>/dev/null
```

Identify the tech stack by scanning for config files (package.json, requirements.txt, pyproject.toml, go.mod, Cargo.toml, pom.xml, etc.) and read them to understand the project.

## Step 3: Deep Analysis

Spawn **parallel Task agents** (subagent_type: general-purpose, model: sonnet) for each analysis track. Provide each agent with the repo path, tech stack info, and relevant file lists.

**Agent 1 — Code Health:**
- Dead code: unused exports, unreachable branches, commented-out code blocks
- Duplicated logic that should be consolidated
- Inconsistent patterns (e.g., mixing async styles, inconsistent error handling)
- Missing error handling at system boundaries
- TODO/FIXME/HACK comments that indicate unresolved issues
- Dependency health: outdated packages with known vulnerabilities, unused dependencies
- Type safety gaps (any types, missing null checks, unsafe casts)

**Agent 2 — Architecture & Structure:**
- Circular dependencies between modules
- Modules with too many responsibilities (god objects/files)
- Missing or inadequate test coverage for critical paths
- Configuration issues (hardcoded values, missing env var validation)
- API contracts: inconsistent response formats, missing validation
- Database: missing migrations, schema drift risks, missing indexes
- Build/CI configuration issues

**Agent 3 — Security & Operations:**
- Secrets or credentials in code or config files
- SQL injection, XSS, command injection, or path traversal risks
- Missing rate limiting, auth checks, or input sanitization
- Unsafe deserialization or file operations
- Missing logging for security-relevant events
- Docker/deployment config issues (running as root, exposed ports, missing health checks)
- Missing or misconfigured CORS, CSP, or security headers

## Step 4: Branch-Specific Analysis (if not --all)

For the target branch, additionally:

```bash
# Diff against base
BASE=$(git merge-base HEAD main 2>/dev/null || git merge-base HEAD master)
git diff $BASE..HEAD --stat
git diff $BASE..HEAD
```

Review all changes on this branch (excluding merge commits) for:
- Incomplete implementations (partial features, missing edge cases)
- Introduced bugs or regressions
- Tests that cover the changes adequately

## Step 5: Compile Diagnosis and Save Report

Compile findings into the report format below. **Save it to `docs/reports/YYYY-MM-DD-repo-doctor.md`** (use today's date). Create `docs/reports/` if it doesn't exist.

Present findings as a numbered checklist so the user can reference items by number when deciding what to fix.

```markdown
# Repository Health Report — {repo-name}

**Date**: {YYYY-MM-DD}
**Branch**: {current/target branch}
**Tech stack**: {detected stack}
**Last commit**: {date and message}

---

## Critical Issues (fix now)

Issues that could cause production incidents, data loss, or security breaches.

- [ ] 1. {file:line} — {what} — {why it matters}

## Warnings (fix soon)

Tech debt, performance risks, or quality issues that will compound over time.

- [ ] 2. {file:line} — {what} — {why it matters}

## Suggestions (improve when possible)

Opportunities to improve code quality, developer experience, or maintainability.

- [ ] 3. {file:line} — {what} — {suggested approach}

## Branch Health

- **Active branches**: N (list with last activity date)
- **Stale branches** (>30 days): N (recommend cleanup)
- **Branches with conflicts**: N
- **Uncommitted changes**: {list files}

## Summary

{2-3 sentence overall assessment — is this repo healthy? What's the biggest risk?}
```

After saving, tell the user: **"Report saved to `docs/reports/YYYY-MM-DD-repo-doctor.md`. Fix what you want, check items off, come back to it anytime."**

**Rules:**
- Be specific — include file paths and line numbers for every finding
- Prioritize by impact: production risks first, DX improvements last
- Don't flag style preferences or subjective design choices
- If the repo is in good shape, say so — don't manufacture issues
- For each finding: **what**, **why it matters**, **how to fix it**
- Number every finding so the user can reference them easily
