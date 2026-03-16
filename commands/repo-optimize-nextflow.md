---
description: Multi-pass Nextflow DSL2 pipeline optimization. Analyzes workflows, subworkflows, modules, and lib/ utilities for channel pattern duplication, embedded script quality, container consistency, meta pattern compliance, and more. Tailored for Nextflow's unique constraints (no linter, no standard test framework, process isolation, channel semantics).
---

# /repo-optimize-nextflow — Nextflow DSL2 Pipeline Optimization

Optimize a Nextflow DSL2 pipeline repository (or scoped subdirectory) using a two-agent architecture designed for Nextflow's unique language constraints.

Unlike generic `/repo-optimize` which targets Python/TS/Go/Rust, this command understands:
- Nextflow channel semantics (scatter-gather, groupTuple, join, combine, branch)
- The workflow → subworkflow → module → lib/ hierarchy
- Process isolation (can't share script variables between processes)
- Embedded Python/Bash heredoc scripts inside process blocks
- Container directives, meta patterns, version emission, resource labels
- Stub blocks as the Nextflow equivalent of characterization tests

## Arguments

The user may optionally provide:
- **Path scope** — a subdirectory to limit the scan (e.g., `/repo-optimize-nextflow subworkflows/`)
- **Category focus** — one or more categories to prioritize (e.g., `/repo-optimize-nextflow --focus channel-duplication,embedded-scripts`)
- **Max passes** — cap the number of passes (e.g., `/repo-optimize-nextflow --passes 3`)
- **Skip stubs** — skip the stub block backfill phase (e.g., `/repo-optimize-nextflow --skip-stubs`)
- **Stubs only** — only add missing stub blocks, no optimization (e.g., `/repo-optimize-nextflow --stubs-only`)

If no arguments are given, scan the entire working directory with defaults.

## Optimization Categories

| # | Category | What It Catches |
|---|----------|----------------|
| 1 | **Channel pattern duplication** | Repeated scatter/gather, groupTuple, join patterns |
| 2 | **Embedded script quality** | Python/Bash heredocs with resource leaks, poor error handling; `script:` vs `shell:` misuse |
| 3 | **Process duplication** | Near-identical processes differing by 1-2 parameters |
| 4 | **Container consistency** | Mismatched versions, missing Elvis fallbacks, missing `procps` |
| 5 | **Meta pattern violations** | Channels dropping `meta.id` during transformations |
| 6 | **Version emission gaps** | Processes missing `versions.yml`, broken aggregation |
| 7 | **Resource label accuracy** | Wrong labels for actual workload; missing dynamic retry scaling |
| 8 | **Module import hygiene** | Unused includes, missing aliases |
| 9 | **Stub block completeness** | Missing stubs preventing `-stub-run` validation |
| 10 | **Documentation staleness** | READMEs, headers, CLAUDE.md out of sync with files |
| 11 | **Directive ordering** | Process directives not in canonical order (official Nextflow docs) |
| 12 | **Error strategy gaps** | Resource-intensive processes without `errorStrategy`; missing OOM retry (exit 137-140) |
| 13 | **Cache-safety violations** | Global var mutation in closures; non-deterministic input pairing; modified input files |
| 14 | **Input validation gaps** | Missing `checkIfExists`, `ifEmpty`, arity; index-based output access (`out[0]`) |

## Orchestration Instructions

**Follow these steps exactly. Each optimization pass MUST be a separate agent invocation.**

### Step 0: Pre-Filter (Orchestrator does this directly)

Before spawning any agents, inventory the Nextflow source files:

```bash
# Find all .nf and .groovy source files with metadata
find {SCOPE_DIR} -name '*.nf' -o -name '*.groovy' | while read f; do
    lines=$(wc -l < "$f" 2>/dev/null || echo 0)
    workflows=$(grep -c '^workflow ' "$f" 2>/dev/null || echo 0)
    processes=$(grep -c '^process ' "$f" 2>/dev/null || echo 0)
    includes=$(grep -c '^include ' "$f" 2>/dev/null || echo 0)
    echo "$lines wf=$workflows proc=$processes inc=$includes $f"
done | sort -rn
```

**Filter rules:**
- SKIP files <= 20 lines (too small to optimize)
- SKIP `_template.nf` files (reference templates, not production)
- SKIP pure config files (`conf/*.config`) unless >100 lines
- SKIP `nextflow.config` unless explicitly scoped
- KEEP all `.nf` files with workflow or process definitions
- KEEP `lib/*.groovy` files > 20 lines
- KEEP `bin/` scripts only if they are referenced by processes in scope

Also check for Nextflow availability and test infrastructure:

```bash
# Nextflow version
nextflow -version 2>&1 | head -3

# Check for nf-test
which nf-test 2>/dev/null && echo "nf-test available" || echo "nf-test not available"

# Check for test profile
grep -l "test" conf/*.config 2>/dev/null && echo "test profile found" || echo "no test profile"

# Check for stub blocks across processes
grep -r "stub:" --include="*.nf" -l . | wc -l
total_processes=$(grep -r "^process " --include="*.nf" -l . | wc -l)
echo "Stub coverage: $stub_files / $total_processes process files"
```

### Step 1: Spawn the Nextflow Optimizer Planner

Spawn a `nextflow-optimizer-planner` agent (or use the `Plan` agent type) with this prompt:

> Analyze the Nextflow DSL2 repository at `{WORKING_DIR}` and create a multi-pass optimization plan.
> **Scope:** {PATH_SCOPE or "entire repository"}
> **Category focus:** {CATEGORIES or "all categories, planner's discretion"}
> **Max passes:** {MAX_PASSES or "6"}
>
> **Pre-filtered candidate files:**
> {CANDIDATE_LIST from Step 0 — path, lines, workflows, processes, includes}
>
> **Infrastructure:**
> - Nextflow version: {version}
> - nf-test: {available/not available}
> - Test profile: {found/not found}
> - Stub coverage: {N}/{M} process files have stubs
>
> Score files across all 14 Nextflow-specific categories (channel duplication, embedded script
> quality, container consistency, meta compliance, directive ordering, error strategies,
> cache-safety, input validation, etc.), classify as STRUCTURAL or COSMETIC, and group into
> passes respecting the workflow → subworkflow → module hierarchy.

Wait for the planner to return. Parse its output.

If the planner returns 0 passes, inform the user and stop.

### Step 2: Stub Block Backfill (if needed)

Parse the planner's "Stub Block Backfill Needed" section. If processes are listed that lack stubs AND the user did not pass `--skip-stubs`:

For each process missing a stub, the orchestrator (you) should add stub blocks directly. Stub blocks follow a predictable pattern:

```groovy
stub:
"""
touch ${meta.id}.output_file_1 ${meta.id}.output_file_2
cat <<-END_VERSIONS > versions.yml
"${task.process}":
    tool-name: stub
END_VERSIONS
"""
```

The stub block must create every file listed in the process's `output:` block.

**Rules:**
- Read the process's `output:` block to determine which files to create
- Use `touch` for regular files, `mkdir -p` for directories
- Always include the `versions.yml` emission
- Use `echo '{...}'` for JSON outputs to create valid stubs

If the user passed `--stubs-only`: add stub blocks and stop. Do not proceed to optimization.

### Step 3: Execute Each Optimization Pass

For each pass from the planner's output, spawn a **separate** `nextflow-optimizer` agent with this prompt:

> You are executing Nextflow optimization pass {PASS_NUMBER} of {TOTAL_PASSES}.
>
> **Working directory:** {WORKING_DIR}
> **Target files for this pass:**
> {LIST_OF_FILES}
>
> **Focus areas (prioritize these, but check all 14 Nextflow categories):**
> {FOCUS_AREAS}
>
> **Planner notes:**
> {PLANNER_NOTES}
>
> **Cross-file observations from planner:**
> {CROSS_FILE_OBSERVATIONS or "None"}
>
> **Priority score:** {SCORE}/100
>
> **Project conventions:**
> - Meta pattern: tuple(val(meta), ...) with meta.id
> - Container pattern: params.x_container ?: 'default:tag'
> - Resource labels: process_single, process_low, process_medium, process_high, process_gpu
> - Version emission: path "versions.yml", emit: versions
> - Lib/ auto-loading: {lib files available}
>
> Read each target file in full, verify usages before removals, apply optimizations
> following Nextflow-specific guidelines, and return results in the specified format.

**Rules:**
- Each pass MUST be a separate agent invocation (fresh context)
- Run passes sequentially (later passes may depend on earlier edits)
- If a pass reports a stub-run failure caused by its changes, STOP and report

### Step 4: Post-Optimization Verification

After all passes complete, run validation:

```bash
# Syntax validation via stub-run (if test profile and Docker available)
nextflow run main.nf -stub-run -profile test,docker 2>&1 | tail -30

# If no test profile, validate individual workflows
for wf in workflows/*.nf; do
    echo "=== Validating $(basename $wf) ==="
    nextflow run "$wf" --help 2>&1 | head -5
done

# Check for broken includes
for f in $(find . -name '*.nf' -not -path './.nextflow/*'); do
    grep "include {" "$f" 2>/dev/null | while IFS= read -r line; do
        name=$(echo "$line" | sed 's/.*include { //' | sed 's/ }.*//' | sed 's/ as .*//')
        if ! grep -q "$name" "$f" 2>/dev/null || [ "$(grep -c "$name" "$f")" -lt 2 ]; then
            echo "WARNING: $name may be unused in $f"
        fi
    done
done

# Verify all processes have stub blocks
for f in $(grep -rl '^process ' --include='*.nf' .); do
    process_names=$(grep '^process ' "$f" | awk '{print $2}')
    for pname in $process_names; do
        if ! grep -q 'stub:' "$f"; then
            echo "MISSING STUB: $pname in $f"
        fi
    done
done
```

### Step 5: Summarize Results

Present a final summary to the user:

```markdown
## Nextflow Optimization Complete

**Scope:** {scope} — {files scanned} files scanned, {files optimized} files optimized
**Passes executed:** {N} of {planned} planned
**Priority range:** {highest} → {lowest} (out of 100)

### Stub Backfill (Phase 2)
- Stubs added: {N} processes
- Files modified: {list}
{Or: "Skipped — all processes have stub blocks" / "Skipped — user passed --skip-stubs"}

### All Optimizations (Phase 3)
{Merged list from all passes, grouped by category}

| Category | Count | Files |
|----------|-------|-------|
| Channel pattern duplication | {N} | {files} |
| Embedded script quality | {N} | {files} |
| ...

### Verification
- Stub-run: PASS/FAIL/SKIPPED
- Include check: PASS/FAIL ({N} includes verified)
- Stub coverage: {before}% → {after}%

### Files Modified
{Deduplicated list — production files and stubs}

### Files Skipped
{Summary with reasons — template, config, too small, clean, stub-only}
```

## When to Use

- After scaffolding new workflows or subworkflows (clean up boilerplate)
- Periodic pipeline hygiene (quarterly)
- Before tagging a production release
- After porting workflows from another Nextflow project (e.g., genomixflow → genetic-pipeline-dsl)
- When adding a new phasing backend, pipeline stage, or module that follows existing patterns

## When NOT to Use

- On `conf/*.config` files (these are declarative configuration, not logic)
- On generated files (e.g., DAG SVGs, trace files)
- On active development branches where files are changing rapidly
- When the pipeline has no stub blocks at all (run with `--stubs-only` first)
- On `nextflow.config` (parameter declarations are a design decision, not an optimization target)
