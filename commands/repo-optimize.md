---
description: Multi-pass code optimization across an entire repository (or subdirectory). Pre-filters files, partitions large codebases into blocks, plans per-block passes, backfills targeted tests, then optimizes. Scales from 1K to 15K+ LOC.
---

# /repo-optimize - Whole-Repository Multi-Pass Optimization

Optimize a repository or subdirectory using a token-efficient multi-phase architecture that scales to large codebases.

For small codebases (<3K LOC): single-block flow — plan all → test all → optimize all.
For large codebases (3K+ LOC): multi-block flow — partition by domain, then each block gets a fresh plan → test → optimize cycle with higher fidelity.

## Arguments

- **Path scope** — subdirectory to limit scan (e.g., `/repo-optimize src/pipelines/`)
- **Category focus** — categories to prioritize (e.g., `/repo-optimize --focus typing,dead-code`)
- **Max passes** — cap total passes across all blocks (e.g., `/repo-optimize --passes 3`)
- **Skip tests** — skip test backfill (e.g., `/repo-optimize --skip-tests`)
- **Tests only** — only backfill tests (e.g., `/repo-optimize --tests-only`)

No arguments = scan entire working directory with defaults.

## Architecture: Single-Block vs Multi-Block

```
Pre-filter → Partition → [Block 1: Plan → Test → Optimize] → [Block 2: ...] → Final Verify
```

**Single block** (≤3K LOC, ≤25 candidates): One planner sees everything. Best for small modules.
**Multi-block** (>3K LOC or >25 candidates): Each block gets a fresh planner scoped to one domain. Each planner has deeper context on fewer files, producing better plans. Inter-block verification catches cross-domain regressions.

## Token Efficiency Principles

- Pre-filter trivial files before any agent reads anything
- Partition large scopes so each planner reads ≤25 files
- Only backfill tests for files receiving **structural** changes with <50% coverage
- Target 2-3 passes per block (4-6 total), not 8-10
- Exclude files scoring below 40
- Keep orchestrator lean — 2-3 line summaries per pass, brief block reports

## Orchestration Instructions

**Follow these steps exactly. Each optimization pass MUST be a separate agent invocation.**

### Step 0: Pre-Filter (Orchestrator does this directly)

Before spawning any agents, cheaply identify candidate files:

```bash
# Get file sizes and basic complexity for all source files
find {SCOPE_DIR} -name '*.py' -o -name '*.ts' -o -name '*.tsx' -o -name '*.js' \
  -o -name '*.rs' -o -name '*.go' -o -name '*.java' | while read f; do
    lines=$(wc -l < "$f" 2>/dev/null || echo 0)
    defs=$(grep -c 'def \|class \|fn \|func \|function ' "$f" 2>/dev/null || echo 0)
    dir=$(dirname "$f")
    echo "$lines $defs $dir $f"
done | sort -rn
```

**Filter rules:**
- SKIP files ≤30 lines (too small to optimize)
- SKIP `__init__.py` / `__init__.ts` / `mod.rs` unless >50 lines
- SKIP files with 0 function/class definitions (pure data/config)
- KEEP everything else as candidates

Also run coverage upfront (if tooling available):

```bash
# Python:
pytest --cov={SOURCE_PKG} --cov-report=term-missing -q 2>&1 | tail -80
# Go: go test -coverprofile=coverage.out ./... && go tool cover -func=coverage.out
# Rust: cargo llvm-cov --text
```

### Step 0.5: Partition into Blocks

Count total candidate LOC and files from Step 0. Then partition:

| Candidates | Total LOC | Blocks | Strategy |
|------------|-----------|--------|----------|
| ≤25 files | ≤3K | 1 | Single block, all candidates |
| 26-50 files | 3-8K | 2 | Split by top-level package/domain |
| 51-80 files | 8-15K | 3 | Split by package, merge small packages |
| >80 files | >15K | 4 | Split by package; suggest scoping to subdirectory |

**Partitioning rules:**
1. **Group by package/directory** — files in the same directory go in the same block
2. **Dependency order** — if package A imports from package B, put B in an earlier block so its shared utilities are ready when A is optimized
3. **Roughly equal LOC** — blocks should be within ±40% of each other in total LOC
4. **Never split a package** across blocks — keep all files from `data_sources/` together
5. **Foundation block first** — if there are `utils/`, `common/`, or `config/` packages, put them in Block 1 (shared utilities created here are used by later blocks)

**Example partition (from our CVA run — 67 files, ~13.6K LOC, 40 candidates after pre-filter):**

| Block | Domain | Packages | ~Files | ~LOC |
|-------|--------|----------|--------|------|
| 1 | Foundation | `config/`, `utils/`, `common/` | 8 | 1,100 |
| 2 | Data Layer | `data_sources/`, `refs/` | 12 | 3,500 |
| 3 | Core Logic | `enrich/`, `processing/` | 14 | 4,200 |
| 4 | Output | `report/`, `orchestration/`, `cli*` | 10 | 3,500 |

### Step 1-4: Execute Each Block

For each block, run this complete cycle:

#### Step 1: Plan (per block)

Spawn the planner agent scoped to this block's files:

> Analyze these files and create a multi-pass optimization plan.
> **Block {B} of {TOTAL_BLOCKS}: {BLOCK_NAME}**
> **Scope:** {BLOCK_PACKAGES}
> **Max passes for this block:** {2-3}
>
> **Candidate files for this block:**
> {CANDIDATE_LIST — file path, line count, def count}
>
> **Coverage data:**
> {COVERAGE_OUTPUT, or "not available"}
>
> **Context from prior blocks (if any):**
> {Brief summary of shared utilities created in earlier blocks, e.g.,
>  "Block 1 created utils/cache.py with is_stale(). Block 2 adopted it in 3 data source modules."}
>
> Score files, classify STRUCTURAL vs COSMETIC, group into 2-3 passes.

The planner for Block 2+ receives a brief summary of what earlier blocks changed. This lets it plan around shared utilities without re-reading those files.

#### Step 2: Test Backfill (per block, if needed)

**Only backfill STRUCTURAL files with <50% coverage** from this block's plan.

Spawn test writer agents (parallel if multiple files):

> Backfill characterization tests for structural optimization targets.
> **Block {B}: {BLOCK_NAME}**
> **Files:** {STRUCTURAL files with <50% coverage in this block}
> **Source dir:** {SOURCE_DIR}
> Focus on functions flagged for structural changes in the planner notes.

Skip if:
- No structural targets need tests in this block
- User passed `--skip-tests`
- Planner said "Test backfill can be skipped"

#### Step 3: Optimize (per block)

For each pass in this block's plan, spawn a separate optimizer agent:

> Optimization pass {N} of {BLOCK_PASSES} (Block {B}: {BLOCK_NAME}).
> **Working dir:** {WORKING_DIR}
> **Test command:** {TEST_CMD}
> **Lint command:** {LINT_CMD}
> **Target files:** {FILES}
> **Focus areas:** {FOCUS}
> **Planner notes:** {NOTES}
> **Cross-file observations:** {CROSS_FILE or "None"}
> **Priority:** {SCORE}/100
>
> Read targets, verify usages before removals, apply optimizations, run verification.

Rules:
- Each pass = separate agent (fresh context)
- Sequential within a block (passes may depend on each other)
- Track only 2-3 line summaries in the orchestrator

#### Step 4: Inter-Block Verification

After each block completes, run the **full** test suite (not just the block's tests):

```bash
# Python: pytest tests/ -x -q --tb=short
# Go: go test ./...
# Rust: cargo test --workspace
```

This catches cross-domain regressions (e.g., Block 2 changed a data source that Block 3's processing module depends on).

**If tests fail after a block:** STOP. Report the failure. Do not proceed to the next block. The user must decide whether to fix or revert.

**Between blocks:** Give the user a brief progress report:

```
Block 1 complete: 4 files optimized (config/paths.py -79 lines, shared is_stale() created).
Tests: 1197 passed. Proceeding to Block 2 (Data Layer).
```

### Step 5: Final Verification

After all blocks complete:

```bash
# Full test suite
pytest tests/ -x -q 2>&1 | tail -20
# Linting
ruff check .
# Coverage (optional)
pytest --cov={PKG} --cov-report=term 2>&1 | tail -5
```

### Step 6: Summarize Results

```markdown
## Repo Optimization Complete

**Scope:** {scope} — {files scanned} scanned, {files optimized} optimized
**Blocks:** {B} ({block names})
**Total passes:** {N} across {B} blocks
**Token efficiency:** {candidates}/{total files} pre-filtered, {blocks} planning cycles

### Per-Block Summary
| Block | Domain | Files Optimized | Tests Added | Key Changes |
|-------|--------|----------------|-------------|-------------|
| 1 | Foundation | 4 | 0 | Shared is_stale(), paths.py -79 lines |
| 2 | Data Layer | 6 | 89 | Adopted shared utils, asdict() |
| ... | ... | ... | ... | ... |

### Optimizations by Category
{Merged list from all blocks/passes}

### Verification
- Lint: PASS/FAIL
- Tests: PASS/FAIL ({N} passed)
- Coverage: {before}% → {after}%

### Files Modified
{All production + test files across all blocks}
```

## Pass Budget Per Block

Each block should produce **2-3 optimization passes** (not more):

| Block Size | Candidates | Passes | Rationale |
|------------|-----------|--------|-----------|
| Small (<1.5K LOC) | ≤8 | 1-2 | One pass may suffice |
| Medium (1.5-4K LOC) | 9-15 | 2-3 | Standard |
| Large (>4K LOC) | >15 | 3 | Max per block; split the block if more needed |

**Total passes across all blocks** should be 4-8 for a typical 10-15K LOC codebase.

## Score Threshold

- **0-39**: EXCLUDE — clean code, not worth optimizer context
- **40-60**: Include in merged passes
- **61-100**: Priority targets

## Optimization Categories

| # | Category | What It Catches |
|---|----------|----------------|
| 1 | **Redundancy** | Duplicate logic, repeated patterns |
| 2 | **Overcomplications** | Unnecessary abstractions, over-engineering |
| 3 | **Resource efficiency** | Wasteful allocations, inefficient data structures |
| 4 | **Type quality** | Weak typing, `Any` usage, missing annotations |
| 5 | **Duplication** | Copy-paste code, near-identical functions |
| 6 | **Long-cuts** | Roundabout solutions (manual loop vs `sum()`, `any()`) |
| 7 | **Dead code** | Unused imports, unreachable branches, stale utilities |
| 8 | **Naming** | Ambiguous names, inconsistent conventions |

## When to Use

- Periodic codebase hygiene (monthly/quarterly)
- Before a major release
- After inheriting or forking a project
- Onboarding to a new codebase

## When NOT to Use

- Test suite is failing (fix tests first)
- Generated or vendored code
- Active feature development on same files
- Very large monorepos without scoping to a subdirectory (>80 candidates even after filtering)
