# Profile Health — ECC Fork Setup

This is the [Profile Health](https://github.com/Profile-Health-Inc) fork of [everything-claude-code](https://github.com/affaan-m/everything-claude-code). It includes custom agents and commands built for our team on top of the upstream ECC plugin.

## What's Added (on top of upstream v1.8.0)

| Type | Name | Description |
|------|------|-------------|
| Agent | `optimizer-planner` | Read-only agent that analyzes `git diff HEAD~1` and produces a multi-pass optimization plan |
| Agent | `optimizer` | Read-write agent that executes a single optimization pass across 8 categories |
| Agent | `repo-optimizer-planner` | Read-only repo-wide analysis with per-file quality findings and history-aware scoring |
| Agent | `repo-optimizer` | Read-write repo-wide optimization with before/after quality reporting per file |
| Agent | `nextflow-optimizer-planner` | Read-only Nextflow DSL2 optimization planner (channel patterns, containers, meta compliance) |
| Agent | `nextflow-optimizer` | Read-write Nextflow DSL2 optimizer for workflows, subworkflows, and modules |
| Agent | `repo-test-writer` | Generates targeted tests for under-tested code paths |
| Command | `/optimize` | Multi-pass code optimization for changes from the last commit |
| Command | `/repo-optimize` | Repo-wide multi-pass optimization with persistent history |
| Command | `/repo-optimize-nextflow` | Nextflow DSL2 pipeline optimization across 14 categories |

## Installation

### Fresh install (no ECC yet)

In Claude Code, run:

```
/plugin marketplace add Profile-Health-Inc/everything-claude-code
/plugin install everything-claude-code@everything-claude-code
```

### Switching from upstream ECC

If you already have the original `affaan-m/everything-claude-code` installed, pick **one** of the two methods below.

#### Option A — Re-point the existing clone (recommended)

```bash
cd ~/.claude/plugins/marketplaces/everything-claude-code
git remote set-url origin git@github.com:Profile-Health-Inc/everything-claude-code.git
git fetch origin
git reset --hard origin/main
```

#### Option B — Delete and re-clone

```bash
rm -rf ~/.claude/plugins/marketplaces/everything-claude-code
git clone git@github.com:Profile-Health-Inc/everything-claude-code.git \
  ~/.claude/plugins/marketplaces/everything-claude-code
```

Then, for **both** options:

1. **Clear the plugin cache** so it rebuilds from the fork:

   ```bash
   rm -rf ~/.claude/plugins/cache/everything-claude-code
   ```

2. **Update the marketplace source** in `~/.claude/plugins/known_marketplaces.json`:

   Inside the `"everything-claude-code"` object, change the nested `source.repo` value:
   ```json
   "everything-claude-code": {
     "source": {
       "source": "github",
       "repo": "Profile-Health-Inc/everything-claude-code"
     },
     ...
   }
   ```

3. **Remove the stale plugin entry** from `~/.claude/plugins/installed_plugins.json`:

   Delete the `"everything-claude-code@everything-claude-code"` key inside `"plugins"` so Claude Code re-installs from the fork on next launch.

4. **Restart Claude Code.** The fork's version (including `/optimize`) will be loaded.

## Staying Up to Date

Pull the latest changes from the fork:

```bash
cd ~/.claude/plugins/marketplaces/everything-claude-code
git pull origin main
rm -rf ~/.claude/plugins/cache/everything-claude-code
```

Then restart Claude Code.

## Syncing with Upstream

To pull in updates from the original ECC repo:

```bash
cd ~/.claude/plugins/marketplaces/everything-claude-code
git remote add upstream https://github.com/affaan-m/everything-claude-code.git  # if not already added
git fetch upstream
git rebase upstream/main  # rebase custom commits on top of latest upstream
# Resolve any conflicts, then force-push the feat branch
git push origin HEAD:feat/repo-optimize-commands --force-with-lease
```