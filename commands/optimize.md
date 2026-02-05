---
description: Multi-pass code optimization for changes from the last commit. Analyzes scope, plans passes for context efficiency, then optimizes redundancy, overcomplications, typing, duplication, and more.
---

# /optimize - Multi-Pass Code Optimization

Optimize code changes from the last commit using a two-agent architecture that maximizes reasoning quality by giving each optimization pass a fresh context window.

## How It Works

1. **Plan** - A read-only planner agent analyzes the diff and creates a pass plan
2. **Execute** - A read-write optimizer agent runs each pass with focused scope
3. **Verify** - Full test suite confirms nothing broke

## Orchestration Instructions

**You MUST follow these steps exactly. Do NOT skip the planning step. Each pass MUST be a separate optimizer agent invocation to preserve fresh context.**

### Step 1: Spawn the Optimizer Planner

Spawn the `everything-claude-code:optimizer-planner` agent with this prompt:

> Analyze the diff from the last commit and create a multi-pass optimization plan.
> Run `git diff HEAD~1 --stat` and `git diff HEAD~1` to understand the scope.
> Group files by logical proximity, size passes for context efficiency, and identify focus areas.
> Return the plan in your specified output format.

Wait for the planner to return. Parse its output to extract each pass definition (files, focus areas, notes).

If the planner returns 0 passes (e.g., only file renames or no meaningful code changes), inform the user and stop.

### Step 2: Execute Each Optimization Pass

For each pass (1 through N) from the planner's output, spawn a **separate** `everything-claude-code:optimizer` agent with this prompt template:

> You are executing optimization pass {PASS_NUMBER} of {TOTAL_PASSES}.
>
> **Target files for this pass:**
> {LIST_OF_FILES}
>
> **Focus areas (prioritize these, but check all 8 categories):**
> {FOCUS_AREAS}
>
> **Planner notes:**
> {PLANNER_NOTES}
>
> Read each target file in full, apply optimizations following your guidelines, run verification (ruff, tests), and return results in your specified output format.

**Important:**
- Each pass MUST be a separate agent invocation (fresh context window)
- Run passes sequentially, not in parallel (later passes may depend on earlier edits)
- If a pass reports test failures, STOP and report the failure to the user before continuing

### Step 3: Post-Optimization Verification

After all passes complete, run the project's full test suite:

```bash
# Detect project type and run appropriate tests
# Python:
PYTHONPATH="$(pwd)/src:$(pwd)" OPENAI_API_KEY=test-key pytest tests/ -x -q

# TypeScript/JavaScript:
# npm test

# Go:
# go test ./...
```

Also run linting:

```bash
# Python:
ruff check .

# TypeScript:
# npx tsc --noEmit
```

### Step 4: Summarize Results

Present a final summary to the user:

```markdown
## Optimization Complete

**Scope:** {files changed} files, {lines modified} lines from last commit
**Passes executed:** {N}

### All Optimizations
{Merged list from all passes, grouped by category}

### Verification
- Linting: PASS/FAIL
- Tests: PASS/FAIL ({N} passed)

### Files Modified
{Deduplicated list of all files touched across all passes}
```

## Optimization Categories Reference

| # | Category | What It Catches |
|---|----------|----------------|
| 1 | **Redundancy** | Duplicate logic, repeated patterns |
| 2 | **Overcomplications** | Unnecessary abstractions, over-engineering |
| 3 | **Resource efficiency** | Wasteful allocations, inefficient data structures |
| 4 | **Type quality** | Weak typing, `Any` usage, missing annotations |
| 5 | **Duplication** | Copy-paste code, near-identical functions |
| 6 | **Long-cuts** | Roundabout solutions (manual loop vs `sum()`, `any()`) |
| 7 | **Dead code** | Unused imports, unreachable branches |
| 8 | **Naming** | Ambiguous names, inconsistent conventions |

## When to Use

- After committing a feature or fix, to polish the code
- Before opening a PR, as a final quality pass
- After a large refactor, to catch leftover cruft

## When NOT to Use

- On uncommitted changes (commit first, then optimize)
- On trivial commits (single-line typo fixes)
- On generated or vendored code
- When the test suite is already failing